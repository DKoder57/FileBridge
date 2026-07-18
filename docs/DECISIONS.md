# Decisões técnicas — FileBridge

## Decisões já tomadas

### O que o app é (e não é)

- É uma ferramenta **P2P, efêmera, baseada em sessão** para arquivos e mensagens
- **Não é** chat persistente, armazenamento em nuvem ou plataforma de mensagens geral
- Usuários-alvo: devs que trabalham com VMs, pessoas que precisam de transferência rápida entre dispositivos sem configuração
- Distribuição: gratuito na Google Play + Microsoft Store, com coleta de feedback no próprio app (sem monetização por enquanto)

### Ideias descartadas

| Ideia | Motivo |
|---|---|
| CLI em Go | Desnecessário — Flutter roda em Linux desktop também, e o caso de uso da VM pode usar o app Flutter direto se houver display |
| Tailscale / WireGuard | Não embutível em app de loja |
| Supabase para sinalização | Projeto Supabase do autor está pausado/em risco; quer separação de infraestrutura |
| Node.js / Express server | Pesado demais pro requisito de "extremamente leve" |
| Código QR | Adiado pro roadmap (não é MVP) |
| Chat persistente | Contra a intenção de design de privacidade/efemeridade |
| Contas / login | Explicitamente não desejado — sem contas, sem lista de contatos |

## Decisões resolvidas (2026-07-18)

### 1. Formato do código de sessão

**Decidido:** `ANIMAL-NN` — palavra em maiúsculas de uma lista de animais + 2 dígitos. Ex: `TIGER-42`.

- Exibição em espelho na tela de geração (fonte grande, orientação invertida) — pensado pra quando os dois dispositivos ficam com a tela virada um pro outro durante o pareamento, facilitando a leitura simultânea dos dois lados. **Assunção a confirmar:** se "espelhado" tinha outro sentido em mente, ajustar o design da tela de geração de código.
- Lista de animais fica em `core/session/` como constante, fácil de expandir depois.

### 2. Regras de produção do Firebase

**Decidido:** Firebase armazena **apenas** o código de sessão e os dados de sinalização (SDP/ICE) — nunca nome de usuário, arquivo, mensagem ou qualquer dado pessoal. Plano gratuito (Spark) confirmado, sem cartão de crédito.

Regras de produção devem:
- Permitir **escrita** apenas para criar um novo nó de sessão com um código ainda não existente (sem sobrescrever sessões ativas de outros)
- Exigir um campo de timestamp na criação do nó
- Restringir **leitura** ao código exato da sessão (sem listagem/enumeração de sessões existentes)
- Expiração (TTL): como o plano Spark não roda Cloud Functions agendadas com saída de rede, a limpeza é feita client-side — ao tentar entrar em um código, o cliente verifica o timestamp e trata como expirado (e limpa) se passou do TTL (sugestão: 5 minutos)

### 3. Limite de tamanho de arquivo

**Decidido:** Sem limite artificial de tamanho. A fragmentação (chunking) é adaptativa. Um aviso/guarda técnica só entra em cena se o tamanho do arquivo puder causar problema real de memória ou desempenho no dispositivo (principalmente mobile) — não como restrição de produto, só como proteção técnica.

### 5. Empacotamento para Windows Store

**Decidido:** Fazer a integração com MSIX. Avaliação do autor: não deve ser difícil nem trabalhoso. Deixa de ser uma questão em aberto e vira item normal do roadmap de lançamento.

### 6. Colisão de códigos de sessão

**Decidido:** Ao gerar/entrar numa sessão, o usuário digita um apelido local (não salvo, não sincronizado, não é conta/login — só usado naquele momento). Esse apelido participa da composição/verificação do código, reduzindo a chance de colisão entre usuários não relacionados. Continua sem contas: o apelido não persiste em disco nem em nenhum backend.

## Questões ainda em aberto

### 4. Fallback quando o P2P via WebRTC falha

Sem estratégia definida ainda. Em redes restritivas, a conexão direta pode falhar mesmo com STUN. `flutter_webrtc` suporta servidores TURN como fallback, mas TURN tem custo — o que conflita com a restrição de zero orçamento para servidores. Precisa de decisão futura: aceitar que a sessão simplesmente falhe nesses casos (mensagem de erro clara ao usuário), ou buscar TURN gratuito/de baixo custo mais adiante.
