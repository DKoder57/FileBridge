# Decisões Técnicas

Registro de decisões importantes do projeto, para evitar perda de contexto e facilitar manutenção futura.

---

Data: 2026-07-18
Decisão: Não haverá contas ou login no FileBridge.
Motivo: Manter o app o mais simples e privado possível — pareamento via código de sessão é suficiente para o caso de uso, sem exigir cadastro, senha ou lista de contatos.
Alternativas consideradas:

* Login com e-mail/senha
* Login social (Google/Apple)
* Perfil local persistente no dispositivo

---

Data: 2026-07-18
Decisão: Sinalização via Firebase Realtime Database (plano gratuito Spark), usada apenas para o handshake WebRTC.
Motivo: Dois dispositivos em redes diferentes não conseguem se encontrar sem um intermediário (problema de NAT). O Firebase resolve isso sem custo, mesmo em escala, pois os dados de sinalização são mínimos e descartados após ~1-2 segundos.
Alternativas consideradas:

* Supabase (projeto do autor já pausado/em risco, quer separação de infraestrutura)
* Servidor de relay próprio (custo de servidor)
* Tailscale/WireGuard (não embutível em app de loja)
* P2P puro sem sinalização (tecnicamente impossível entre redes diferentes)

---

Data: 2026-07-18
Decisão: Formato do código de sessão é `ANIMAL-NN` (ex: `TIGER-42`), com exibição espelhada na tela de geração.
Motivo: Precisa ser curto, legível e fácil de comunicar verbalmente ou visualmente entre duas pessoas. A exibição espelhada facilita a leitura quando os dois dispositivos ficam com a tela virada um para o outro durante o pareamento.
Alternativas consideradas:

* Código numérico de 6 dígitos
* Formato adjetivo-substantivo-número

---

Data: 2026-07-18
Decisão: Firebase armazena apenas o código de sessão, timestamp e dados de sinalização (SDP/ICE) — nunca dados pessoais, arquivos ou mensagens. Regras de produção restringem escrita à criação de sessões novas, restringem leitura ao código exato (sem listagem), e a expiração (TTL, ~5 min) é verificada client-side.
Motivo: Manter a garantia de privacidade do produto mesmo em produção, e evitar custo de Cloud Functions agendadas, que exigem o plano pago (Blaze).
Alternativas consideradas:

* Cloud Functions agendadas para limpeza automática (exige plano Blaze, fora do orçamento zero)
* Regras de leitura/escrita abertas (mantidas apenas em desenvolvimento, inseguras para produção)

---

Data: 2026-07-18
Decisão: Sem limite artificial de tamanho de arquivo. Fragmentação adaptativa via DataChannel, com aviso (não bloqueio) apenas quando o tamanho puder causar problema real de memória/desempenho no dispositivo.
Motivo: O produto não deve impor uma restrição arbitrária ao usuário; a única preocupação legítima é a estabilidade técnica do app em dispositivos com pouca memória.
Alternativas consideradas:

* Limite fixo de tamanho (ex: 100MB) independente do dispositivo
* Bloqueio hard acima de um limiar técnico

---

Data: 2026-07-18
Decisão: Fazer a integração de empacotamento MSIX para publicação na Microsoft Store.
Motivo: Passo necessário para builds Flutter Windows em lojas; avaliado pelo autor como simples de implementar, sem justificar adiamento.
Alternativas consideradas:

* Distribuir o app Windows apenas via instalador próprio, fora da Microsoft Store

---

Data: 2026-07-18
Decisão: Resolver colisão de código de sessão com um apelido local (não salvo, não sincronizado, não é conta) digitado a cada sessão, que participa da composição/verificação do código.
Motivo: Reduz a chance de dois usuários não relacionados gerarem o mesmo código simultaneamente, sem violar a decisão de não ter contas — o apelido nunca persiste em disco ou backend.
Alternativas consideradas:

* Códigos mais longos (menor probabilidade de colisão, mas menos amigável de digitar)
* Checagem de unicidade centralizada no servidor sem envolver o usuário
* Limpeza baseada em TTL apenas (não resolve colisão simultânea, só libera códigos expirados)

---

Data: 2026-07-18
Decisão: Estratégia de fallback para quando o P2P via WebRTC falha (redes restritivas onde STUN não é suficiente) ainda não foi definida.
Motivo: TURN resolveria o problema tecnicamente, mas tem custo recorrente, o que conflita com a restrição de zero orçamento para servidores. Falta decidir entre aceitar a falha com mensagem clara ao usuário ou buscar um provedor de TURN gratuito/baixo custo.
Alternativas consideradas:

* Aceitar falha da sessão com mensagem de erro clara (sem custo, pior experiência em redes restritivas)
* Servidor TURN próprio ou de terceiros (resolve o problema, mas tem custo)

> Nota (2026-08-30): enquanto esta decisão não for tomada, a implementação de compartilhamento de tela (F5) não deve começar. Vídeo contínuo sofre muito mais com P2P instável do que a troca pontual de arquivos, então construir F5 sobre uma base de conectividade ainda indefinida arrisca retrabalho.

---

Data: 2026-08-30
Decisão: Usar servidores STUN públicos (ex: `stun.l.google.com`) como configuração padrão do `RTCPeerConnection` para resolução de candidatos ICE, independente da decisão de fallback TURN acima (ainda em aberto).
Motivo: Sem STUN, o handshake WebRTC só é confiável dentro da mesma rede local, o que não cobre o caso de uso de origem (VM ↔ host em redes distintas). STUN público não tem custo e resolve a maior parte dos cenários de NAT antes de precisar de TURN.
Alternativas consideradas:

* Depender apenas de rede local no MVP (não cobre o caso de uso de origem do projeto)
* Rodar STUN próprio (custo/manutenção desnecessários havendo opções públicas gratuitas)

---

Data: 2026-08-30
Decisão: Controlar o envio de arquivos pelo `bufferedAmount` do `RTCDataChannel`, pausando o loop de escrita quando o buffer de saída ultrapassar um limite definido e retomando via `onBufferedAmountLow`.
Motivo: Sem controle de fluxo, o app pode enfileirar dados mais rápido do que a rede consegue escoar, causando crescimento descontrolado de memória em arquivos grandes. É um problema de arquitetura, não de linguagem — precisa ser resolvido independentemente de qualquer decisão futura sobre Rust.
Alternativas consideradas:

* Confiar apenas em chunks pequenos, sem controle de fluxo explícito (não resolve em conexões lentas)
* Limitar a taxa de envio por tempo fixo (menos preciso que reagir ao estado real do buffer)

---

Data: 2026-08-30
Decisão: Mover leitura de arquivo, cálculo de hash e chunking para uma `Isolate` dedicada (ou `compute()`), fora da isolate principal do Dart.
Motivo: Esse processamento é CPU-bound; rodando na isolate principal, trava a UI durante transferências grandes. Resolve o problema de responsividade sem precisar trocar de linguagem/stack neste estágio do projeto.
Alternativas consideradas:

* Manter tudo na isolate principal (mais simples de implementar, mas trava a UI em arquivos grandes)
* Reescrever essa camada em Rust via FFI desde já (prematuro sem dados de baseline — ver decisão de avaliação de Rust abaixo)

---

Data: 2026-08-30
Decisão: Uma queda momentânea de rede tenta reconexão via ICE restart automático antes de encerrar a sessão, retomando a transferência a partir do último chunk confirmado.
Motivo: Reiniciar a sessão inteira e reenviar o arquivo do zero após uma queda breve de rede é uma experiência ruim e desnecessária, especialmente em transferências grandes.
Alternativas consideradas:

* Encerrar a sessão imediatamente em qualquer queda de conexão (mais simples de implementar, pior experiência)
* Retomar a conexão mas reenviando o arquivo inteiro (desperdiça banda e tempo do usuário)

---

Data: 2026-08-30
Decisão: Nenhuma otimização de desempenho mais profunda (ex: mover trechos do core para Rust via FFI, adicionar compressão de arquivo) será adotada sem antes medir throughput e uso de memória do app em cenários reais (arquivos pequenos, médios e grandes).
Motivo: Evitar reescrever partes do app com base em suposição sobre onde está o gargalo real. A camada de rede do FileBridge já roda nativa (`libwebrtc` via `flutter_webrtc`); o ganho de trocar de stack só se justifica se houver gargalo medido em trabalho CPU-bound (hash, compressão), não em rede.
Alternativas consideradas:

* Migrar partes do core para Rust preventivamente, sem medição prévia
* Decidir por intuição/pressão de "o app está pesado", sem dados concretos

---

Data: 2026-08-30
Decisão: Ainda não decidido. Avaliação formal de mover chunking/hashing/compressão para Rust via `flutter_rust_bridge` fica condicionada aos números coletados na instrumentação de métricas (ver decisão acima).
Motivo: Rust só traz ganho real em trabalho CPU-bound (hash, compressão); a camada de rede (WebRTC) já roda nativa em C++ via `libwebrtc`, então trocar de stack sem medir não teria benefício comprovado e ainda adicionaria complexidade de build (toolchain Rust + cross-compilation por plataforma).
Alternativas consideradas:

* Adotar Rust preventivamente em todo o `core` (custo de build desproporcional a um ganho não comprovado)
* Descartar Rust por completo desde já (fecha a porta para um ganho real caso hashing/compressão de arquivos grandes se mostre um gargalo genuíno)

---

Data: 2026-08-30
Decisão: Ainda não decidido. Compressão (ex: gzip) antes do envio será testada de forma condicional — pulando tipos de arquivo tipicamente já comprimidos (zip, jpg, mp4) — e só ativada por padrão se o ganho líquido (tempo de compressão + transferência menor) for consistentemente positivo nos testes.
Motivo: Comprimir todo arquivo indiscriminadamente pode piorar o tempo total em arquivos já comprimidos, gastando CPU sem ganho real de banda.
Alternativas consideradas:

* Comprimir todo arquivo sempre (simples, mas pode piorar desempenho em arquivos já comprimidos)
* Nunca comprimir (mais simples, mas deixa banda na mesa em arquivos que se beneficiariam, ex: texto/logs)
