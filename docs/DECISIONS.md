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

## Questões em aberto

Ainda não resolvidas — precisam de decisão antes ou durante a implementação:

1. **Formato do código de sessão** — `TIGER-42`? 6 dígitos? `adjetivo-substantivo-número`? Qualquer string curta e legível serve, mas a lógica de geração precisa de um formato específico pra implementar.

2. **Regras de produção do Firebase** — hoje o repo mostra regras de dev/teste. Regras de produção devem restringir escrita a novos códigos de sessão apenas, e definir um TTL de expiração. Estrutura exata ainda não decidida.

3. **Limite de tamanho de arquivo** — WebRTC DataChannel aguenta arquivos grandes, mas a estratégia de fragmentação importa em mobile. Sem limite definido ainda.

4. **Fallback quando o P2P via WebRTC falha** — em redes restritivas, a conexão direta pode falhar mesmo com STUN. `flutter_webrtc` suporta servidores TURN como fallback, mas TURN tem custo. Sem estratégia de fallback definida.

5. **Empacotamento pra Windows Store** — builds Flutter Windows exigem empacotamento MSIX pra loja. Ainda não discutido; adiciona um passo ao build.

6. **Colisão de códigos de sessão** — se dois usuários não relacionados gerarem o mesmo código ao mesmo tempo, eles se conectariam um ao outro. Precisa de estratégia de resistência a colisão (códigos mais longos, checagem de unicidade no servidor, ou limpeza baseada em TTL).
