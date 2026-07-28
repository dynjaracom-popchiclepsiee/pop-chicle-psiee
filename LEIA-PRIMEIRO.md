# Publicação no Netlify

Este pacote é uma cópia independente do portal. Ele não altera o site atualmente publicado.

## O que você precisa

- Uma conta no Netlify.
- Uma conta no Supabase para guardar escolas, publicações, fotos, PDFs e vídeos.

O Netlify hospeda o site. O Supabase guarda os dados e arquivos. Isso é necessário porque o portal é dinâmico; um HTML isolado não preserva publicações entre navegadores.

## 1. Preparar o Supabase

1. Crie um projeto em https://supabase.com/dashboard.
2. Abra **SQL Editor**, clique em **New query**, copie todo o arquivo `supabase/schema.sql` e execute.
3. Em **Project Settings > API**, copie:
   - Project URL
   - `service_role` key
4. Em **Project Settings > Database**, copie a **Connection string** do modo Transaction pooler.
5. Em **Storage > school-files > Configuration**, confirme o limite desejado. O arquivo SQL solicita 1 GB, mas o limite efetivo depende do seu plano do Supabase.

## 2. Colocar no GitHub

1. Crie um repositório vazio no GitHub.
2. Extraia este ZIP.
3. Envie todos os arquivos da pasta extraída para o repositório, incluindo `netlify.toml`.

## 3. Publicar no Netlify

1. No Netlify, clique em **Add new project > Import an existing project**.
2. Escolha o repositório do GitHub.
3. O Netlify reconhecerá:
   - Build command: `npm run build`
   - Publish directory: `.next`
4. Antes de publicar, abra **Environment variables** e crie:

| Variável | Valor |
| --- | --- |
| `DATABASE_URL` | Connection string do Supabase |
| `SUPABASE_URL` | Project URL do Supabase |
| `NEXT_PUBLIC_SUPABASE_URL` | A mesma Project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Chave `service_role` do Supabase |
| `ADMIN_USERNAME` | Seu usuário de administração |
| `ADMIN_PASSWORD` | Sua senha de administração |
| `ADMIN_SESSION_TOKEN` | Uma sequência aleatória longa, com pelo menos 40 caracteres |

5. Clique em **Deploy**.

## 4. Entrar

- Abra o endereço gerado pelo Netlify.
- Entre como administradora com `ADMIN_USERNAME` e `ADMIN_PASSWORD`.
- Crie as escolas e seus códigos normalmente.

## Importante

- Não publique nem envie o arquivo `.env` para o GitHub. O pacote contém somente `.env.example`, sem senhas.
- O Netlify e o Supabase possuem limites de plano. Nenhuma hospedagem legítima pode prometer armazenamento e funcionamento ilimitados para sempre.
- Os dados do site atual não migram automaticamente, pois pertencem ao banco da hospedagem atual. Este pacote começa com um banco novo.
- Depois de publicar, use o domínio do Netlify como endereço oficial. Assim, cancelar ou mudar o plano do ChatGPT não altera essa nova cópia.
