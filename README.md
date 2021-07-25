# Deploy de aplicações NodeJs na AWS  
## Crie uma conta  
Será preciso usar um cartão de crédito. Escolha a modalidade free e lembre-se de desativar todos os serviços depois de usar o AWS ou você ter que pagar uma “graninha” ai.
* Acesse [https://portal.aws.amazon.com/](https://portal.aws.amazon.com/)
* Escolha a opção "Basic support free"
* Acesse o console
* Digite e-mail e senha Root

## Criando a instância/Servidor  
* Escola a opção EC2 (Servidor Linux)
* Clique em "Lauch instance"
* Escolha "Amazon 2 Linux"
* Escolha o free "t2 micro"
* Clique em "Review And Launch"

## Configurando questões de acesso a aplicação   
* Vá em "Edit security groups"
* Adicione uma porta http, para acessarmos a aplicação pelo navegador
* Clique em "Review and Launch"
* Clique em launch

## Chaves de acesso  
Agora vamos criar uma chave que usaremos para acessar o servidor
* Clique em "Create a new key pair"
* Escolha um nome
* Faça o download e salve em um local seguro

No Windows você precisará manipularas permissões
* Clique no arquivo
* Vá em propriedades
* Segurança
* Desabilite a Herança
* Remova todos os usuários com exceção do Admin
* Todas as operações que envolverem essa chave precisarão ser feitas como usuário admin

## Continuando  
* Clique em "Lauch instances"
* Clique em "View instances"
* Espere a instancia executar (running)
* Clique no id da instancia
* Vai em networking(redes)
* Lembre se dos dados de "Public IPv4 address"
* Lembre dos dados de "Public IPv4 DNS" (vamos usar para conectar ao servidor)

## Acessando o servidor  
Use o comando abaixo para conectar ao servidor
```shell
ssh -i ${chave} ec2-user@${PublicIpv4Dns}
```

Exemplo de como fica
```shell
ssh -i "chave.pem" ec2-user@ec2-1-111-111-111.us-east-2.compute.amazonaws.com
```

* Clique em "yes"
* Agora você tem acesso ao servidor

## Criando a pasta do projeto  
Iremos criar uma pasta e dar permissões completa para acessarmos ela via outros programas como o Filezila

* Escolha uma pasta e acesse ela. "cd nomePasta"
* Crie uma pasta para a aplicação "sudo mkdir meuapp"
* De permissões completas para ela "sudo chmod -R 777 meuapp/"


-----------------------------

## Instalando o Node e o nginx1  
Confira [nessa página](https://docs.aws.amazon.com/pt_br/sdk-for-javascript/v2/developer-guide/setting-up-node-on-ec2-instance.html) para obter a forma mais atual de instalar o Node na AWS

Comandos para realizar a instalação
```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
. ~/.nvm/nvm.sh
nvm install node
node -v
node -e "console.log('Running Node.js ' + process.version)"
```

Comandos para instalar o nginx1
```shell
sudo amazon-linux-extras install nginx1
```   

## Rodando o nginx.  
Só contextualizando, usaremos ele como reverse proxy, o aplicativo NodeJs rodará em uma porta, exemplo 8080 e usaremos o nginx para redirecionar a porta padrão do servidor para a porta 8080.

```shell
# Comando para iniciar o serviço Nginx
sudo service nginx start

# Comando para ver o status do serviço Nginx
sudo service nginx status

# Comando para parar o serviço Nginx
sudo service nginx stop

# Comando para reiniciar o serviço Nginx
sudo service nginx restart
```

Acesse o IP (Somente IP) "Public IPv4 address" para ver o nginx rodando.


## Configurando regras  
Agora vamos criar uma regra para permitir o acesso a porta do nosso App. Se o seu app for porta 5000, use ela, no meu caso era 8080.
* Vá em "instance"
* Clique me "Security"
* Clique em "Security groups"
* Clique em "Edit inbound rules"
* Clique em "Add rule"
* Escola a opção "Custom tcp"
* Escolha a porta do seu app, no meu caso 8080
* Escolha para todos os endereços "0.0.0.0/0.0"
* Clique em "save rules"

-------------------------------

## Configurando o filezila  
Aviso importante, você pode fazer o processo pelo terminal, usar o git clone ou até algo mais incrementado, mas se você não tiver familiarizada com o terminal, o filezila pode ser uma ótima opção.

* Abra o filezila
* Clique em "File"
* Clique em "Site Manager"
* Clique em "new site"
* Escolha um nome para o site
* Em Protocolo escolha SFTP
* Como host coloque o "Public IPv4 DNS" completo
* Como usuário coloque "Normal"
* No campo usuário coloque "ec2-user"
* Em password deixe vazio
* Clique em ok

Ok, agora vamos configurar a chave, sim, precisamos daquela chave que baixamos início para acessar o servidor usando o filezila
* Clique em "Editar"
* Clique em "Configuracoes"
* Clique em "SFTP"
* Clique em "add keyt file"
* Escolha a chave
* Clique em OK
* Em file > Site Manager > tente se conectar.
* Deve dar certo e você conseguirá acessar a pasta que criou e deu as permissões 777.
  

## Suba o seu projeto  
Seja usando o git, filezila ou outra solução, chegou a hora de subir o app. Não suba o node_modules por favor!
* No terminal, acesse a pasta que você criou e colocou seus arquivos e rode o comando "npm i" para instalar as dependências
* Rode sua aplicação com o comando "node index.js"
* Acesse o IP em "Public IPv4 address" com a porta 8080 (porta do seu app)
* Ok, você deverá ver sua aplicação rodando.

Importante, se você fechar o terminal, sua aplicação cai, é igual se você estivesse fazendo na sua máquina

-------------------

## Usando o pm2  
Para fazermos o app ser persistente usaremos uma biblioteca chamada PM2
* Use o comando "npm install pm2@latest -g" para instalar ela de forma global
* Rode o comando "pm2 start index.js"
* Pronto, a aplicação rodará mesmo se você fechar o terminal

Mais dicas:

```shell
# Ver os status
pm2 status

# Ver os logs em tempo rela
pm2 log

# Parar o pm2
pm2 stop index.js
```


## Fazendo o reverse proxy  
Ok, agora que a aplicação já roda sozinha, precisamos fazer o reverse proxy para conseguirmos acessar a aplicação através do da porta 80 (padrão da internet)

Edite o arquivo "nginx.conf" com o comando
```shell
sudo nano /etc/nginx/nginx.conf
```

Comente o trecho por completo
```textplan
    server {
        listen       80;
         .....
        root         /usr/share/nginx/html;
...
```

Substitua por esse trecho
```shell
server {
  location / {
    proxy_pass http://localhost:8080;
    proxy_http_version 1.1;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connecion "upgrade";
    proxy_set_header Host $host;
    proxy_cache_bypass $http_updagr;
  }
}
```
* Reinicie o serviço e rode a aplicação node novamente
* Digite o comando "sudo service nginx restart"
* Acesse IP "Public IPv4 address", sem usar a porta do APP. Deve dar certo!


## Apontando um DNS para o servidor  
Ok, agora vamos a parte de linkar o servidor/ip a um DNS com https. Esse trecho ainda preciso testar, pois preciso comprar um domínio para alguma aplicação futuramente.

Você precisa ter um domínio comprado em alguma plataforma.

* Vamos usar a cloudflare para apontar o DNS para o servidor.
* Crie uma conta
* Clique em "Adicionar um site"
* Escolha o plano free
* Aponte o ip do domínio para sua aplicação web escolhendo APENAS o plano TIPO A
* Clique para ativar
* Serão gerados dois nameservers, na plataforma que você comprou os domínios aponte eles.
* Clique em continuar
* Escola a opção HTTPS
* Marque para sempre redirecionar para HTTPS
* Pode demorar até 24h para o DNS propagar.
* E pronto, aplicação publicada!

Digite "exit" no terminal para fechar a conexão.
