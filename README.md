#!/bin/bash

# ----------------------------------------------------------------------------------
# 1. ATUALIZAÇÃO E INSTALAÇÃO DE PACOTES
# ----------------------------------------------------------------------------------
echo "Iniciando o provisionamento do Servidor Web Seguro (HTTPS)..."

sudo apt update -y
sudo apt install apache2 -y

# Instala o OpenSSL, necessário para gerar o certificado, e o módulo SSL do Apache.
echo "Instalando OpenSSL e o módulo SSL do Apache..."
sudo apt install openssl -y
sudo a2enmod ssl
echo "Módulo SSL habilitado."
echo "----------------------------------------------------------------------------------"

# ----------------------------------------------------------------------------------
# 2. CRIAÇÃO DO CERTIFICADO AUTOASSINADO
# ----------------------------------------------------------------------------------
echo "Gerando o Certificado SSL Autoassinado (Self-Signed Certificate)..."

# Cria um diretório para armazenar as chaves e certificados
sudo mkdir -p /etc/apache2/ssl/

# Gera a chave privada (server.key) e o certificado (server.crt)
# -nodes: Não criptografa a chave privada
# -x509: Cria um certificado autoassinado
# -days 365: Validade do certificado (1 ano)
# -subj: Preenche as informações de forma não interativa (CN=Common Name é o mais importante)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/apache2/ssl/apache.key -out /etc/apache2/ssl/apache.crt -subj "/C=BR/ST=SP/L=SaoPaulo/O=Projeto Infra/OU=TI/CN=meudominio.com"

echo "Certificado e chave criados em /etc/apache2/ssl/."
echo "----------------------------------------------------------------------------------"

# ----------------------------------------------------------------------------------
# 3. CONFIGURAÇÃO DO VIRTUAL HOST SSL
# ----------------------------------------------------------------------------------
echo "Configurando o Virtual Host SSL..."

# Copia e adapta o arquivo de configuração SSL padrão do Apache (default-ssl.conf)
sudo cp /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/projeto.conf

# Modifica o arquivo de configuração para apontar para o novo certificado
# Usa sed para substituir o caminho padrão pelos caminhos do nosso certificado.
sudo sed -i 's|SSLCertificateFile.*|SSLCertificateFile /etc/apache2/ssl/apache.crt|' /etc/apache2/sites-available/projeto.conf
sudo sed -i 's|SSLCertificateKeyFile.*|SSLCertificateKeyFile /etc/apache2/ssl/apache.key|' /etc/apache2/sites-available/projeto.conf

# Habilita o novo site e o site SSL padrão
sudo a2ensite projeto.conf
# Remove o site padrão para evitar conflitos (opcional, mas limpa a config)
sudo a2dissite 000-default.conf
echo "Virtual Host SSL configurado e habilitado."
echo "----------------------------------------------------------------------------------"

# ----------------------------------------------------------------------------------
# 4. CRIAÇÃO DA PÁGINA WEB E REINÍCIO
# ----------------------------------------------------------------------------------

echo "Criando uma página web de teste..."
# Garante que o diretório /var/www/html esteja limpo
sudo rm -f /var/www/html/index.html

# Cria uma nova página index.html simples
cat <<EOT | sudo tee /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>Servidor Web SEGURO</title>
    <meta charset="utf-8">
</head>
<body>
    <h1>🔒 Servidor Web HTTPS Provisionado!</h1>
    <p>A conexão está criptografada com um certificado autoassinado (HTTPS).</p>
    <p>Lembre-se: em produção, use Let's Encrypt ou um certificado de uma CA confiável!</p>
</body>
</html>
EOT

echo "Reiniciando o serviço Apache para aplicar as configurações SSL..."
sudo systemctl restart apache2
echo "----------------------------------------------------------------------------------"

# ----------------------------------------------------------------------------------
# 5. CONFIGURAÇÃO DE FIREWALL (UFW)
# ----------------------------------------------------------------------------------
echo "Configurando o firewall para acesso HTTPS (porta 443)..."

# Permite o tráfego na porta 443 (HTTPS)
sudo ufw allow 'Apache Full'

echo "Porta 443 (HTTPS) e Porta 80 (HTTP) liberadas no firewall (UFW)."
echo "----------------------------------------------------------------------------------"

echo "✅ Fim da configuração do Servidor Web Seguro!"
echo "Acesse https://[IP_DA_MAQUINA] no navegador. Você verá um aviso de segurança, pois o certificado é autoassinado."
