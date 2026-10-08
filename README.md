# Jailbreak para Saraiva Lev (Modelo CYBOY4F-SA)

Este repositório fornece os arquivos e instruções para:

1. **Habilitar acesso root via SSH** no e-reader Saraiva Lev (jailbreak).
2. **Instalar o LEV_OS**, uma interface própria que abre no lugar da Loja: busca e navegação na internet em modo leitor, gerenciador de arquivos, downloads e painel do sistema.
3. **Recuperar o aparelho travado em "Iniciando"**, com um cartão SD inserido nele.

Os arquivos ficam na aba **Releases**.

## Especificações do dispositivo testado

- Modelo: CYBOY4F-SA (fabricado por Bookeen/Cybook), "Lev com luz"
- Anatel: 4185-13-2101
- Processador: ARMv7l (Allwinner A13), 182 MB de RAM
- Kernel: 3.0.8+
- Wi-Fi: Realtek RTL8188EU

Tudo aqui foi testado em **um único aparelho**. Em outro firmware ou revisão, pode se comportar diferente.

---

## Parte 1 — Jailbreak (SSH root)

### Conteúdo

| Arquivo | Para que serve |
|---|---|
| **`CybUpdate.bin`** | **É este que você instala.** Pacote de atualização com SSH (Dropbear) incluído. A versão interna do sistema de arquivos root foi definida como 999 para forçar a sobrescrita (flash) independente da versão atual do firmware. |
| `jailbreak_6_3_2350_cybft_2350.bin` | O mesmo pacote com a versão interna original (12). Fica como referência; o atualizador pode recusá-lo em aparelhos com firmware mais recente. |
| `dropbear-2025.89.tar.bz2` | Código-fonte do Dropbear 2025.89, para quem quiser compilar um servidor SSH mais novo. Não é necessário para o jailbreak. |

Os dois pacotes têm bootloader, kernel e sistema de arquivos idênticos; a única diferença é o número de versão.

### Instalação

1. Conecte o Lev ao computador pelo cabo USB.
2. Copie `CybUpdate.bin` para a **raiz** do armazenamento do aparelho (fora de qualquer pasta).
3. Ejete com segurança.
4. O Lev detecta o pacote sozinho. Confirme a instalação e aguarde o reinício. Não interrompa.

### Acesso por SSH

O Dropbear do aparelho usa protocolos antigos, desativados por padrão no OpenSSH atual. É preciso habilitá-los na linha de comando:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa -c aes128-cbc root@192.168.1.XXX
```

- Usuário: `root`
- Senha padrão: `lev`

Atalho opcional (`.bashrc` ou `.zshrc`):

```bash
alias ssh-lev="ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa -c aes128-cbc root@192.168.1.XXX"
```

**Como achar o IP:** o Lev não mostra o próprio IP. Veja na lista de aparelhos do roteador ou varra a rede atrás de quem tem a porta 22 aberta. O rádio só fica ligado enquanto algo usa a rede, então deixe a Loja aberta na tela enquanto procura.

**Erro de certificado SSL na Loja/navegador:** quase sempre é o relógio do aparelho em data antiga (ele volta para 2015 depois de uma recuperação). Acerte por SSH:

```sh
date -u -s "2026-10-08 12:00:00"   # data e hora atuais, em UTC
hwclock -w -u
```

---

## Parte 2 — LEV_OS v1.1 (interface no lugar da Loja)

### O que é

O atalho **Loja** do Lev passa a abrir uma interface local, servida pelo próprio aparelho:

| Tela | O que faz |
|---|---|
| **Início** | Campo de busca/endereço, hora, bateria e estado do Wi-Fi |
| **Internet** | Busca (DuckDuckGo) e navegação em **modo leitor**; favoritos editáveis |
| **Arquivos** | Navega no armazenamento e no sistema inteiro; abre, mostra como texto e apaga |
| **Downloads** | Baixa por endereço (HTTP e HTTPS) para a pasta `Downloads`; livros baixados aparecem na biblioteca |
| **Sistema** | Memória, rede, acertar relógio pela internet, ligar SSH, manter o Wi-Fi ligado, programas rodando, teste de botões, reiniciar |

### Por que "modo leitor"

O navegador embutido do Lev é um WebKit de 2014 com OpenSSL 1.0.1: não abre a maior parte dos sites atuais. No LEV_OS, quem busca a página é um `curl` atual (estático, incluído no pacote); um script em `awk` remove scripts e estilos e devolve HTML simples, com os links e os formulários de busca passando pelo mesmo caminho.

Tempos medidos no aparelho: busca ~2 s, página comum 1 a 3 s, artigo grande da Wikipédia (2 MB) ~13 s.

### Navegação por folhas

O navegador do Lev não rola a tela. Por isso cada página é cortada na altura da tela (758×1024) e ganha uma barra embaixo:

- **< ANTERIOR** e **PRÓXIMA >** trocam de folha;
- tocar no número do meio (ex.: `6 / 18`) abre uma grade para ir direto a qualquer folha.

Isso vale para todas as telas: leitor, arquivos, favoritos e sistema.

### Limites conhecidos

- **Sites que são aplicativos em JavaScript** (ChatGPT, claude.ai, Pinterest, redes sociais) **não funcionam**, nem no modo leitor nem no navegador original.
- O modo leitor não mostra imagens (só o texto alternativo) e não mantém login em sites.
- Formulários são sempre enviados por GET; os que exigem POST podem falhar.
- O link **[abrir sem o leitor]** entrega o endereço ao navegador original, que na maioria dos sites atuais responde "Erro de certificado SSL". Só serve para os poucos sites que ele ainda abre.
- O Wi-Fi do sistema original oscila; o leitor tenta de novo sozinho (até três vezes) e a tela de erro tem o botão TENTAR DE NOVO.
- O botão físico continua com a função original (voltar aos livros); ele não chega ao navegador como tecla.
- Vídeo é inviável na tela e-ink.

### Como funciona

- O firmware executa `/mnt/fat/boordr` (arquivo `boordr` na raiz do armazenamento), se existir, no lugar do leitor. O gancho deste pacote dispara o LEV_OS em segundo plano e **sempre** termina chamando o leitor original.
- `LEV_OS/start.sh` copia os scripts para a memória (`/tmp/levos`), liga a interface de loopback (o firmware a deixa desligada) e sobe o `inetd` do BusyBox escutando **somente em 127.0.0.1:8080**.
- `LEV_OS/httpd.sh` é o servidor: um processo de shell por pedido.
- `LEV_OS/leitor.awk` é o conversor do modo leitor.
- `LEV_LAUNCHER/index.html` é a página que a Loja abre: um modo básico que pula para `http://127.0.0.1:8080/` quando o serviço local responde.

Nada é gravado na NAND, no rootfs ou no bootloader. A única alteração fora do armazenamento USB é o arquivo `/priv/private/rescueurl` (o endereço que a Loja abre), feita no passo 3 abaixo.

### Instalação

Pré-requisito: jailbreak feito e SSH funcionando (Parte 1). Tenha o cartão SD de recuperação pronto antes de começar (Parte 3).

**1. Copie os arquivos.** Com o Lev no USB, extraia `LEV_OS_v1.1.zip` e copie para a **raiz** do armazenamento:

```
boordr
LEV_OS/
LEV_LAUNCHER/
```

Se já existir um arquivo `boordr` na raiz (de outra modificação), não sobrescreva: acrescente nele o bloco do LEV_OS, antes da linha `exec`.

**2. Ejete com segurança e tire o cabo.** O serviço local sobe quando o leitor é relançado. Com o cabo ligado e o armazenamento exportado ao computador, o aparelho não enxerga os arquivos.

**3. Aponte a Loja para a interface.** Por SSH, com o cabo fora:

```sh
cp /priv/private/rescueurl /mnt/fat/LEV_LAUNCHER/rescueurl.original
printf 'file:///mnt/fat/LEV_LAUNCHER/index.html\n' > /priv/private/rescueurl
sync
```

**4. Toque em Loja.** Deve abrir a tela de início do LEV_OS. Se abrir "Lev - modo básico", o serviço local não subiu: toque em "ABRIR A INTERFACE COMPLETA" ou confira por SSH:

```sh
ls /tmp/levos            # deve ter httpd.sh, leitor.awk, inetd.conf, inetd.pid
netstat -ltn | grep 8080 # deve mostrar 127.0.0.1:8080
sh /mnt/fat/LEV_OS/start.sh; echo $?   # 0 = ok
```

### Atualizar da v1 para a v1.1

Com o Lev no USB, copie por cima os arquivos da pasta `LEV_OS` (`start.sh`, `httpd.sh`, `leitor.awk`) e o `LEV_LAUNCHER/index.html`. O `boordr` não mudou. Ejete e tire o cabo; a versão nova entra sozinha. Os favoritos (`LEV_OS/favoritos.txt`) são preservados.

### Histórico

- **v1.1**: navegação por folhas com seletor de folha; modo leitor corrigido para páginas com dados dentro das marcas (Wikipédia); tabelas ajustadas à largura da tela; nova tentativa automática quando o DNS ou a conexão falham; página grande deixou de levar minutos (era o `sed` do BusyBox).
- **v1**: primeira versão.

### Desligar e desinstalar

- **Desligar sem apagar nada:** crie um arquivo vazio chamado `DISABLED` dentro da pasta `LEV_OS` (pelo USB, em qualquer computador). A Loja passa a abrir só o modo básico.
- **Remover o gancho:** apague o arquivo `boordr` da raiz do armazenamento.
- **Devolver a Loja original:** por SSH,

```sh
cp /mnt/fat/LEV_LAUNCHER/rescueurl.original /priv/private/rescueurl
sync
```

Depois disso, as pastas `LEV_OS` e `LEV_LAUNCHER` podem ser apagadas.

### Segurança

- O servidor local roda como root, mas só aceita conexões do próprio aparelho (127.0.0.1). Não fica acessível pela rede.
- O SSH do jailbreak usa a senha padrão `lev` e fica acessível a quem estiver na mesma rede Wi-Fi. Evite redes públicas com o SSH ligado.

### Componentes de terceiros incluídos

- `LEV_OS/bin/curl`: curl 8.22.0 estático para ARMv7 (musl), do projeto [stunnel/static-curl](https://github.com/stunnel/static-curl).
- `LEV_OS/bin/cacert.pem`: certificados raiz da Mozilla, extraídos por [curl.se](https://curl.se/docs/caextract.html).

Os hashes SHA-256 de todos os arquivos estão em `SHA256SUMS`.

---

## Parte 3 — Recuperação de boot (aparelho travado em "Iniciando")

Se o Lev ligar mas ficar preso na tela **"Iniciando"**, dá para recuperá-lo pelo atualizador oficial da Bookeen, usando um **cartão SD inserido no aparelho**. O material completo (guia, relatório, scripts e registros) está no release **Recuperação_de_boot** (`Recuperacao.lev.zip`).

### O que você precisa

- Um cartão SD/microSD **pequeno, antigo e simples**. O que funcionou foi um de **2 GB**; um de 8 GB falhou.
- O arquivo `CybUpdate.OK.bin` do release de recuperação (é idêntico ao `CybUpdate.bin` do jailbreak).
- Um computador para copiar os arquivos para o cartão.

Não se sabe se o cartão de 2 GB funcionou pelo tamanho, pela idade, pelo formato FAT/FAT32 ou pelo controlador. Sabe-se apenas que o de 8 GB falhou e o de 2 GB funcionou. Se o update não iniciar, tente outro cartão pequeno e antigo.

### Preparar o cartão

A raiz do cartão deve ter exatamente dois arquivos:

```
CybUpdate.bin     (cópia do CybUpdate.OK.bin, com este nome)
.update_lock      (arquivo vazio)
```

No PowerShell, supondo o cartão em `F:`:

```powershell
Copy-Item ".\CybUpdate.OK.bin" "F:\CybUpdate.bin" -Force
New-Item -Path "F:\.update_lock" -ItemType File -Force
```

### Rodar a recuperação

1. Ligue o Lev e espere travar em "Iniciando".
2. **Com ele já travado**, insira o cartão.
3. Aguarde a tela oficial de update aparecer.
4. Mantenha o aparelho na energia e não toque em nada.
5. Espere aparecer **BOOKEEN / UPDATE OK** (a tela fica parada enquanto o cartão estiver dentro).
6. Remova o cartão e reinicie o aparelho sem ele.

Durante o update: não remova o cartão nem a energia, não aperte Power, não tente reset físico e não mexa em testpoints.

### Depois de recuperar

- **Neutralize o cartão**, para o update não rodar de novo por acidente: renomeie `CybUpdate.bin` para `CybUpdate.OK.bin` e `.update_lock` para `update_lock.OK`.
- **Acerte o relógio.** O aparelho volta com a data em 2015, o que causa erro de certificado SSL na Loja (veja o comando na Parte 1).
- No aparelho testado, os arquivos do armazenamento (livros e a pasta do LEV_OS) continuaram lá depois da recuperação.

**Recomendação:** deixe esse cartão preparado e guardado antes de instalar qualquer modificação deste repositório.

---

## Créditos

A base técnica do jailbreak foi documentada por br-lemes:
https://www.br-lemes.net/2017/03/ssh-no-lev.html

## Isenção de responsabilidade

Este software é fornecido "no estado em que se encontra", sem garantias de qualquer tipo. Modificar o firmware é um procedimento de risco e pode resultar na perda da garantia ou na inutilização (brick) do aparelho se não for executado corretamente.
