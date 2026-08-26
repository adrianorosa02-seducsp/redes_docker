Descobrindo o Endereço IP
Execute:

getent hosts lab-server
Anote o endereço IP retornado.

Exemplo:

10.x.x.x    lab-server
Depois execute:

ping lab-server
Observe que o nome:

lab-server
é convertido para um endereço IP.

Conceito
Isso demonstra o funcionamento da resolução de nomes dentro da rede Docker.

7. Acesso SSH
Agora vamos acessar o servidor.

Execute:

ssh aluno@lab-server
Na primeira conexão será apresentada uma mensagem semelhante a:

The authenticity of host 'lab-server' can't be established.
Digite:

yes
Quando solicitado, informe a senha fornecida pelo professor.

Após o login, deverá aparecer:

aluno@lab-server:~$
8. Identificando o Servidor
Execute:

hostname
Resultado esperado:

lab-server
Agora:

whoami
Resultado:

aluno
