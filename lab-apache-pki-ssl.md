Aqui tens o relatório técnico completo, detalhado e estruturado em formato Markdown, pronto a ser copiado e guardado no teu repositório do GitHub (por exemplo, com o nome lab-apache-pki-ssl.md).

Este documento consolida toda a jornada técnica, arquitetura de rede, comandos e o processo de resolução de problemas que executámos hoje.
Relatório Técnico de Laboratório: Implementação de Servidor Web Seguro (HTTPS) com Apache, Easy-RSA e PKI Interna

    Data de Execução: 26 de Setembro de 2026

    Autor: Emanuel Reis (Aluno do ENTA (Escola das Novas Tecnologias dos Açores) - CET Cibersegurança)

    Ambiente / Tecnologias: AWS EC2, Ubuntu Desktop Linux (LUXSRV), Windows Server (WINSERVER), Apache2, Easy-RSA, SSL/TLS, DNS/Hosts.

1. Resumo Executivo e Objetivos

Este laboratório teve como finalidade simular uma infraestrutura de cibersegurança e administração de sistemas empresarial de ponta a ponta, integrando ambientes Linux e Windows. Os principais objetivos atingidos foram:

    Configurar regras de perímetro e controlo de tráfego numa cloud pública (AWS).

    Instalar e configurar um servidor web Apache (LUXSRV) em ambiente Linux.

    Criar uma Autoridade de Certificação (CA) interna de raiz e emitir certificados digitais seguros através do Easy-RSA.

    Configurar o acesso seguro por HTTPS (443/TCP) utilizando um domínio simulado de laboratório (www.enta.pt).

    Configurar o sistema operativo cliente (WINSERVER) através da importação de certificados de confiança raiz e mapeamento estático de nomes de domínio.

2. Topologia de Rede e Arquitetura

    Servidor Web / Aplicação (LUXSRV):

        Sistema Operativo: Ubuntu Desktop Linux (hospedado em instância AWS EC2).

        Endereço IP Privado: 172.31.12.125

        Serviços Ativos: Apache2 (Portas 80 e 443), Easy-RSA (/etc/easy-rsa).

    Máquina Cliente / Teste (WINSERVER):

        Sistema Operativo: Windows Server.

        Função: Validação de cliente, importação de root CA no repositório do sistema e testes de navegação web segura.

    Domínio de Laboratório: www.enta.pt

3. Passo a Passo Técnico da Implementação
Fase 1: Configuração de Perímetro (AWS Security Groups)

Para garantir a livre circulação do tráfego web legítimo para os testes de laboratório, foram ajustadas as regras de entrada (Inbound Rules) na consola da AWS:

    HTTP: Porta 80/TCP | Origem: 0.0.0.0/0

    HTTPS: Porta 443/TCP | Origem: 0.0.0.0/0

    SSH: Porta 22/TCP | Origem restrita para administração remota.

Fase 2: Instalação e Configuração do Apache no Linux (LUXSRV)

Acedendo ao terminal Bash da instância Linux, procedeu-se à instalação e ativação dos pacotes base:
Bash

# Atualizar listas de pacotes e instalar o Apache e o Easy-RSA
sudo apt update
sudo apt install apache2 easy-rsa -y

# Ativar o módulo SSL no servidor web
sudo a2enmod ssl

Fase 3: Criação da PKI e Emissão de Certificados (Easy-RSA)

Para garantir a encriptação sem dependência de entidades externas comerciais, gerou-se uma CA própria:
Bash

# Configurar o diretório de trabalho do Easy-RSA
make-cadir ~/easy-rsa
cd ~/easy-rsa

# Inicializar a Infraestrutura de Chaves Públicas e construir a CA
./easyrsa init-pki
./easyrsa build-ca

# Gerar a chave privada e o CSR (Certificate Signing Request) para o servidor web
./easyrsa gen-req enta-server nopass

# Assinar o pedido de certificado utilizando a CA interna recém-criada
./easyrsa sign-req server enta-server

Fase 4: Configuração do Virtual Host Seguro no Apache

    Copiar os certificados gerados para o diretório seguro do Apache:
    Bash

    sudo mkdir -p /etc/apache2/ssl
    sudo cp pki/issued/enta-server.crt /etc/apache2/ssl/
    sudo cp pki/private/enta-server.key /etc/apache2/ssl/
    sudo cp pki/ca.crt /etc/apache2/ssl/

    Criar o ficheiro de configuração do Virtual Host (/etc/apache2/sites-available/enta-ssl.conf):
    Apache

    <VirtualHost *:443>
        ServerName www.enta.pt
        DocumentRoot /var/www/html

        SSLEngine on
        SSLCertificateFile /etc/apache2/ssl/enta-server.crt
        SSLCertificateKeyFile /etc/apache2/ssl/enta-server.key
        SSLCertificateChainFile /etc/apache2/ssl/ca.crt

        <Directory /var/www/html>
            Options Indexes FollowSymLinks
            AllowOverride All
            Require all granted
        </Directory>
    </VirtualHost>

    Ativar o site e reiniciar o serviço:
    Bash

    sudo a2ensite enta-ssl
    sudo systemctl restart apache2

    Validar a escuta de rede nos sockets:
    Bash

    sudo ss -tulnp | grep apache2

    (Resultado esperado: escuta ativa confirmada em *:80 e *:443).

Fase 5: Configuração e Validação no Cliente Windows (WINSERVER)

    Confiança na Raiz (Root CA): O certificado ca.crt foi copiado para o Windows, aberto e instalado em Local Machine na pasta Trusted Root Certification Authorities (Autoridades de Certificação Raiz Confiáveis), eliminando alertas de certificado não fidedigno.

    Resolução de Nomes (Ficheiro Hosts): Para traduzir o domínio fictício www.enta.pt para o IP privado correto da máquina Linux, editou-se o ficheiro em C:\Windows\System32\drivers\etc\hosts (executando o Bloco de Notas como Administrador), adicionando a linha limpa:
    Plaintext

    172.31.12.125    www.enta.pt

4. Troubleshooting (Resolução de Incidentes)

    Incidente: O navegador no WINSERVER devolvia um erro de ligação recusada (Can't reach this page / ERR_CONNECTION_REFUSED) ao tentar aceder a [https://www.enta.pt](https://www.enta.pt).

    Processo de Diagnóstico:

        Estado do Serviço: Verificado com systemctl status apache2 — Servidor a correr ativamente.

        Escuta de Portas: Verificado com ss -tulnp — Porta 443 aberta e operacional.

        Firewall Local: Verificado com ufw status — Inativa (sem bloqueios internos no Linux).

        Causa Raiz Identificada: Ausência de resolução estática do nome de domínio simulado no cliente e necessidade de apontar explicitamente o tráfego para o IP privado da máquina Linux (172.31.12.125).

    Resolução Aplicada: Inclusão correta do registo IP/Domínio no ficheiro hosts do Windows, restabelecendo com sucesso o canal de comunicação.

5. Validação e Conclusão

    Teste Final: Acedeu-se a [https://www.enta.pt](https://www.enta.pt) através do browser no WINSERVER.

    Resultado: O portal carregou com sucesso, exibindo o indicador de Connection is secure (cadeado fechado), com o certificado devidamente validado pela Autoridade de Certificação interna do laboratório.
