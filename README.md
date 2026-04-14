Jailbreak para Saraiva Lev (Modelo CYBOY4F-SA)

Este repositório fornece os arquivos e instruções necessários para habilitar acesso root via SSH no e-reader Saraiva Lev. O pacote foi modificado para contornar falhas de verificação de versão que impediam a instalação em dispositivos com firmwares mais recentes.
Especificações Técnicas do Dispositivo Testado

    Modelo: CYBOY4F-SA (Fabricado por Bookeen/Cybook)

    Anatel: 4185-13-2101

    Processador: ARMv7l (Allwinner A13)

    Versão do Kernel: 3.0.8+

Conteúdo do Repositório

    CybUpdate.bin: Pacote de atualização modificado. A versão interna do sistema de arquivos root foi definida como 999 para forçar a sobrescrita (flash) independente da versão atual do firmware.

    dropbear-2025.89: Binário do servidor SSH Dropbear compilado para arquitetura ARM.

Procedimento de Instalação
1. Transferência dos Arquivos

    Conecte o Lev ao computador via cabo USB.

    Copie o arquivo CybUpdate.bin para o diretório raiz do dispositivo.

        Nota: O arquivo deve estar na pasta principal, fora de qualquer subdiretório.

2. Atualização do Firmware

    Ejete o dispositivo com segurança.

    O sistema do Lev detectará automaticamente o pacote de atualização.

    Confirme a instalação e aguarde o reinício do aparelho. Não interrompa o processo.

Acesso Remoto (SSH)

O servidor SSH (Dropbear) utiliza protocolos de segurança legados que são desativados por padrão em clientes SSH modernos (OpenSSH). Para conectar, é necessário habilitar explicitamente a troca de chaves, o tipo de chave do host e as cifras compatíveis.
Comando de Conexão (Linux/macOS)

Substitua 192.168.1.XXX pelo endereço IP do dispositivo (disponível nas configurações de rede do Lev).
Bash

ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa -c aes128-cbc root@192.168.1.XXX

    Usuário: root

    Senha padrão: lev

Configuração de Atalho (Opcional)

Para simplificar o acesso, adicione um alias ao seu arquivo .bashrc ou .zshrc:
Bash

alias ssh-lev="ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa -c aes128-cbc root@192.168.1.XXX"

Informações Adicionais e Créditos

O projeto original e a base técnica para este jailbreak foram documentados por br-lemes. Para entender o funcionamento detalhado dos scripts internos e a lógica do jailbreak, consulte a documentação original:
https://www.br-lemes.net/2017/03/ssh-no-lev.html

Isenção de Responsabilidade

Este software é fornecido "no estado em que se encontra", sem garantias de qualquer tipo. A modificação do firmware do dispositivo é um procedimento de risco e pode resultar na perda da garantia ou inutilização (brick) do hardware se não for executado corretamente.
