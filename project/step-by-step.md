# Step by Step

primeiro eu vou criar um cluster novo para o projeto a partir do kind-config ja com o nginx instalado.

Depois vou criar um namespace para o projeto

Preciso buildar a minha imagem localmente e depois subir para dentro do kind com o comando `kind load docker-image <nome-da-imagem> --name <nome-do-cluster>`.

E no deployment, preciso colocar imagePullPolicy: Never para o kind usar a imagem local e não tentar baixar do registry.

Lembrando que a app precisa da variavel de ambiente "DATABASE_URL" para funcionar, então preciso criar um secret com esse valor e referenciar ele no deployment.
