# Ingestão de atendimentos (WhatsApp Web e marketplaces)

Supabase `cjwurpumthjumtgjjolt`. Uma linha por conversa em `atendimentos`, chave única **(canal_id, contato_telefone_ou_usuario)**.
Canais (`canais_atendimento.nome`): `B2B Laben`, `Nutríssima`, `Site`, `Shopee`, `TikTok Shop`, `Mercado Livre`.

Campos: status `novo|em_atendimento|aguardando_cliente|aguardando_nos|resolvido`; ultima_msg_de `cliente|nos`;
prioridade `baixa|normal|alta`. Telefone em E.164 sem símbolos (ex.: `5511999998888`); marketplace = usuário/ID do comprador.

## SQL (execute_sql) — upsert da conversa + mensagens
```sql
with c as (select id from canais_atendimento where nome = 'B2B Laben'),
a as (
  insert into atendimentos (canal_id, contato_nome, contato_telefone_ou_usuario, assunto, status,
                            ultima_msg_em, ultima_msg_de, ultima_msg_texto, nao_lidas)
  select c.id, 'Nome do contato', '5511999998888', 'Cotação avelã 3 kg',
         'aguardando_nos', '2026-10-20T09:40:00-03:00', 'cliente', 'Qual o preço do balde?', 2
  from c
  on conflict (canal_id, contato_telefone_ou_usuario) do update set
    contato_nome     = coalesce(excluded.contato_nome, atendimentos.contato_nome),
    assunto          = coalesce(atendimentos.assunto, excluded.assunto),
    ultima_msg_em    = excluded.ultima_msg_em,
    ultima_msg_de    = excluded.ultima_msg_de,
    ultima_msg_texto = excluded.ultima_msg_texto,
    nao_lidas        = excluded.nao_lidas,
    -- status: msg do cliente reabre/pede resposta; msg nossa passa a aguardar cliente
    status = case when excluded.ultima_msg_de = 'cliente' then 'aguardando_nos'
                  when atendimentos.status = 'resolvido' then 'resolvido'
                  else 'aguardando_cliente' end
  returning id
)
insert into atendimento_mensagens (atendimento_id, em, de, texto)
select a.id, m.em::timestamptz, m.de, m.texto from a,
  (values ('2026-10-20T09:40:00-03:00','cliente','Qual o preço do balde?')) as m(em, de, texto)
on conflict (atendimento_id, em, de) do nothing;
```
Não sobrescreva `responsavel`, `proximo_passo`, `proximo_passo_data`, `observacoes` nem `cliente_id`: são editados no painel.

## REST (PostgREST, chave anon do painel)
1. `GET /rest/v1/canais_atendimento?select=id,nome` → pegar `canal_id`.
2. Upsert:
```
POST https://cjwurpumthjumtgjjolt.supabase.co/rest/v1/atendimentos?on_conflict=canal_id,contato_telefone_ou_usuario
apikey / Authorization: Bearer <anon>
Prefer: resolution=merge-duplicates,return=representation
```
```json
{
  "canal_id": "<uuid do canal>",
  "contato_nome": "Nome do contato",
  "contato_telefone_ou_usuario": "5511999998888",
  "assunto": "Cotação avelã 3 kg",
  "status": "aguardando_nos",
  "ultima_msg_em": "2026-10-20T09:40:00-03:00",
  "ultima_msg_de": "cliente",
  "ultima_msg_texto": "Qual o preço do balde?",
  "nao_lidas": 2
}
```
3. Histórico (opcional): `POST /rest/v1/atendimento_mensagens?on_conflict=atendimento_id,em,de` com
`Prefer: resolution=ignore-duplicates` e corpo `[{"atendimento_id":"<id>","em":"...","de":"cliente","texto":"..."}]`.

Regras de status para o agente: última msg do cliente → `aguardando_nos`; última msg nossa → `aguardando_cliente`;
conversa nova → `novo`. Só o painel (humano) marca `resolvido`. O painel destaca espera >2h úteis (seg–sex 8–18h) e >24h.
