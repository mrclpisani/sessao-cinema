---
name: desempenho
description: Roteiro para melhorar o desempenho da Sessão de cinema (compartilhamento de tela por WebRTC/PeerJS em index.html) — mede, acha o gargalo, corrige uma coisa por vez e revisa, chamando as outras skills na ordem certa. Use sempre que o pedido envolver imagem travando, fps baixo, imagem borrada, atraso, som falhando, CPU alta de quem compartilha, qualidade caindo com mais espectadores, página lenta para abrir, ou qualquer "melhorar", "otimizar" ou "deixar mais leve", mesmo que a pessoa não diga a palavra "desempenho".
---

# Desempenho da Sessão de cinema

O app é um arquivo só (`index.html`) que envia a tela direto de navegador para navegador. Quase todo
problema de desempenho dele mora em um de quatro lugares: o PC de quem compartilha, a rede no caminho,
o aparelho de quem assiste, ou o carregamento da página. O trabalho é descobrir qual deles antes de mexer.

O código já carrega decisões medidas (veja os comentários em `preferirCodec`, `limitar` e `ATRASO_MS`).
Mudar sem um número novo troca um ajuste testado por um palpite, e é assim que desempenho piora.

## A ordem

Siga na sequência. Cada passo existe para alimentar o seguinte.

| # | Passo | Skill | Por que nesta posição |
|---|---|---|---|
| 1 | Nomear o sintoma | — | Define o que medir. Se a pessoa já disse, não pergunte de novo. |
| 2 | Medir como está | `/run` | Sem o número de antes não dá para provar melhora depois. |
| 3 | Achar o gargalo | tabela abaixo; `/systematic-debugging` se os números não fecharem | Corrigir o lugar errado não muda nada. |
| 4 | Mudar uma coisa e medir de novo | — | Duas mudanças juntas escondem qual delas funcionou. |
| 5 | Limpar o que foi alterado | `/simplify` | Só faz sentido quando já existe alteração que funcionou. |
| 6 | Revisar a alteração | `/engineering:code-review` | Pega regressão antes de chegar aos amigos. |
| 7 | Revisar segurança, se tocou em CSP, autorização ou `REDE` | `/security-review` | Ganho de desempenho não paga por abrir a sala. |
| 8 | Confirmar com números | `/verification-before-completion` | Fecha com antes e depois, não com "deve ter melhorado". |

Fora da sequência, só de vez em quando:

- `/ponytail:ponytail-audit` — procura peso morto no repositório inteiro. Aqui raramente é o gargalo
  (são dois arquivos), então não comece por ele.
- Plugins de Lighthouse (Perf Profiler, Web Vitals Auditor) — só se a queixa for a página demorando a
  abrir. Eles não enxergam a transmissão.

## Passo 2: como medir

**Subir o app.** A configuração `sessao-cinema` em `.claude/launch.json` serve a pasta em
`http://localhost:8765`.

**Autorização local.** A página só libera aparelhos listados em `autorizados.json`, e o navegador de
teste não está lá. Abra a página, copie o código que ela mostra, acrescente uma linha `TesteLocal` em
`autorizados.json`, recarregue e, ao terminar, desfaça com `git checkout -- autorizados.json`. Essa
linha nunca entra em commit: ela daria acesso à sala para um navegador que não é de ninguém.

**Fonte sintética.** `?teste` na URL troca o seletor de tela por uma imagem gerada (640×360) com som;
`?teste=mudo` simula captura sem áudio. Serve para conferir que a conexão, o codec e a medição
funcionam e para comparar antes e depois de uma mudança no envio. Não serve para medir custo de
codificação: a imagem é pequena e simples demais (cerca de 2 ms por quadro contra 13 ms em 1080p real).

**Números reais.** O seletor de tela do navegador não pode ser automatizado. Para medir um filme de
verdade, peça à pessoa para compartilhar e ler a dica do selo de cada amigo (passar o mouse), ou
colar o trecho abaixo no console e devolver o resultado. O endereço `chrome://webrtc-internals` mostra
os mesmos números em gráfico ao longo do tempo.

Abra a sala em uma aba, copie `linkSala`, abra o link em outra aba e clique em Assistir. Espere uns
10 segundos antes de ler: os primeiros segundos são a rede ainda subindo a taxa.

Na aba de quem compartilha:

```js
await Promise.all([...chamadas].map(async c => {
  const st = [...(await c.peerConnection.getStats()).values()];
  const o = st.find(s => s.type === 'outbound-rtp' && s.kind === 'video') ?? {};
  const r = st.find(s => s.type === 'remote-inbound-rtp' && s.kind === 'video') ?? {};
  const p = st.find(s => s.type === 'candidate-pair' && s.nominated) ?? {};
  return { amigo: c.nome, codec: st.find(s => s.id === o.codecId)?.mimeType,
    altura: o.frameHeight, fps: o.framesPerSecond, msPorQuadro: +(o.totalEncodeTime / o.framesEncoded * 1000).toFixed(1),
    limite: o.qualityLimitationReason, segundosLimitado: o.qualityLimitationDurations,
    alvoMbps: o.targetBitrate / 1e6, redeAguentaMbps: p.availableOutgoingBitrate / 1e6,
    rttMs: r.roundTripTime * 1000, perda: r.fractionLost, nack: o.nackCount, pli: o.pliCount };
}))
```

Na aba de quem assiste:

```js
const st = [...(await chamadaAtual.peerConnection.getStats()).values()];
const i = st.find(s => s.type === 'inbound-rtp' && s.kind === 'video') ?? {};
({ codec: st.find(s => s.id === i.codecId)?.mimeType, altura: i.frameHeight, fps: i.framesPerSecond,
  msPorQuadro: +(i.totalDecodeTime / i.framesDecoded * 1000).toFixed(1), descartados: i.framesDropped,
  travadas: i.freezeCount, segundosTravado: i.totalFreezesDuration, perdidos: i.packetsLost,
  nack: i.nackCount, pli: i.pliCount, bufferMs: Math.round(i.jitterBufferDelay / i.jitterBufferEmittedCount * 1000) })
```

Os contadores são acumulados desde o início da conexão. Para saber o que acontece agora, leia duas
vezes com alguns segundos de intervalo e compare.

## Passo 3: do número ao ajuste

| O que a medição mostra | Gargalo | Onde mexer |
|---|---|---|
| `limite: cpu`, ou `msPorQuadro` de envio perto de 33 | PC de quem compartilha | Codec (`#codec`, `preferirCodec`); `frameRate` e `height` em `capturar`; menos espectadores (`MAX_AMIGOS`) |
| `limite: bandwidth`, `redeAguentaMbps` abaixo da qualidade escolhida | Upload de quem compartilha ou rede até o amigo | Qualidade menor (`limitar`); AV1 rende mais imagem por Mbps, ao custo de CPU |
| Envio bom, mas no amigo `travadas`, `perdidos` ou `nack` subindo | Rede no caminho | `ATRASO_MS` maior; taxa menor |
| Envio bom, mas no amigo `descartados` subindo ou `msPorQuadro` alto | Aparelho do amigo | Codec mais leve de decodificar (VP9 antes de AV1); resolução menor |
| `fps` baixo com `limite: none` | Captura | Conteúdo parado gera poucos quadros, e isso é normal. Confira com vídeo em movimento antes de concluir. Depois: aba em vez de tela inteira |
| Imagem borrada com `fps` bom | Taxa pequena para a resolução | Qualidade maior ou codec mais eficiente. `maintain-resolution` já impede o navegador de encolher a imagem |
| Reações na call chegam antes da cena | `ATRASO_MS` somado ao `rttMs` | Reduzir `ATRASO_MS`, sabendo que a troca é mais travada em rede ruim |
| Não conecta, ou reconecta sem parar | NAT das duas redes | Não é desempenho. Use "Testar minha rede"; a saída é um retransmissor (TURN), que custa dinheiro e é decisão da pessoa |
| Página demora a abrir | Carregamento | `peerjs.min.js` (87 KB, baixado antes de a página reagir); a busca de `autorizados.json` |

Se os números não apontarem para nenhuma linha, ou apontarem para duas que se contradizem, pare de
ajustar e use `/systematic-debugging`.

## Limites que ajuste nenhum remove

Diga-os à pessoa em vez de tentar otimizá-los por conta própria. São decisões de arquitetura e de custo.

- **Uma codificação por espectador.** Cada amigo é uma conexão própria, então a CPU de quem
  compartilha cresce junto com a plateia. Sair disso exige um servidor no meio (SFU).
- **Sem retransmissor.** Redes que não se alcançam direto (4G, CGNAT, VPN) não conectam.
- **Medição vale para a máquina medida.** O comentário em `preferirCodec` fala de um PC específico.
  Em outro hardware a ordem dos codecs pode mudar, e por isso o seletor de codec fica na tela: é o
  botão de calibragem, não o remova para "simplificar".
- **Segurança não entra na troca.** A autorização por desafio, a CSP e o teto de espectadores não se
  afrouxam para ganhar desempenho.

## Passo 8: como relatar

Feche sempre com esta tabela, usando os mesmos campos nas duas colunas e a mesma fonte de vídeo:

```
Sintoma: <o que a pessoa relatou>
Gargalo: <qual linha da tabela, e o número que provou>
Mudança: <arquivo:linha, uma frase>

| Medida           | Antes | Depois |
|------------------|-------|--------|
| fps              |       |        |
| altura           |       |        |
| ms por quadro    |       |        |
| limite           |       |        |
| <a do sintoma>   |       |        |

Medido em: <fonte sintética ou captura real; quantos espectadores; qual máquina>
Não verificado: <o que ficou de fora, por exemplo celular ou rede móvel>
```

Se o depois não for melhor que o antes, desfaça a mudança. Código a mais sem ganho medido é só peso.
