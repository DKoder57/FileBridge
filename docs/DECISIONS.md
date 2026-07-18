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
