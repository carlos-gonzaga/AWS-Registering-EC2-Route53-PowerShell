# Registrando IP do Servidor Windows no Route53 ao Iniciar a instância EC2 na AWS


### Objetivo

Esse script tem a finalidade de fazer com que sua EC2 Windows se registre no Route53 toda vez que for iniciada.

Principal objetivo é economizar com o custo de ElasticIP equanto sua instância estiver desligada.

```
Para instâncias Windows Server 2016 ou superior é possível adicionar o script na configuração de User Data em Advanced Details no terceiro passo de criação da instância EC2, copie o script PowerShell-Script/User-Data.ps1 e edite conforme passo 4 abaixo, e cole na configuração de User Data da instância, com isso, os passos 3 em diante não são necessários.
```



### Requisitos

- PowerShell v3.0
- AWS CLI
- A instância deve ser criada com a opção **Auto-assign Public IP**, selecione essa opção no Step 3 de criação da EC2.
    Caso a instância não esteja com essa configuração, necessário recriar a instância com essa opção habilitada
- Entrada DNS no Route53 a qual o Servidor será vinculado


### Passo a passo

1. Criar Role no IAM para permitir que o servidor altere o Route53\
    Você poderá criar via script do Cloudformation com o yml no diretório **Role**. Especifique o HostedZoneID para permitir alterações apenas em uma zona específica, ou então especifique nenhum parâmetro de entrada, para criar uma Role generica com permissão de alterar todas as Zonas.

    Para verificar o ID da Zona: Entre no Route53 no Console da AWS => Hosted Zone => Selecione a Zona desejada (Não click no domínio) => Irá aparecer na lateral direita as informações da Zona, inclusive o Hosted Zone ID.

    **Permissões Necessárias:**
    ```
    route53:ListHostedZone
    route53:GetHostedZone
    route53:ListResourceRecordSets
    route53:ChangeResourceRecordSets
    ```

    **Criação da Role via Cloudformation**
    ```bash
    aws cloudformation create-stack --stack-name "Update-Route53-Role" --template-body file://Role/Update-Route53-Role.yml --parameters ParameterKey=HostedZoneID,ParameterValue="XXXXXXXXXXXXXX",ParameterKey=DomainName,ParameterValue="mydomain"
    ```


2. Associe a policy a sua instância EC2 para que a mesma tenha permissão\
    *Caso a instância já exista:*\
    No Console da AWS no serviço EC2, selecione a instância => Actions => Instance Settings => Attach/Replace IAM Role => Selecione Dynamic-DNS

    *Na criação da Instância:*\
    No Step 3: Configure Instance Details => Em IAM Role => Selecione a Role Update-Route53-Domain


3. Instale o AWS CLI na instância criada, caso ainda não esteje instalado\
    https://s3.amazonaws.com/aws-cli/AWSCLI64.msi


4. Copie o Script em **PowerShell-Script/route53-set-public-ip.ps1** para o servidor e altere as variáveis com as informações do seu Domínio e Servidor
    
   **test.meudominio.com.br**
                
Variável  | Valor
------------- | -------------
DomainName    |  meudominio.com.br.
Subdomain     |  test
RecordType    |  A
ServerName    |  Verifique o nome do servidor, comando hostname no prompt de comando

> A configuração de ServerName foi adicionada para que caso você crie uma AMI da instância com essas configurações, a configuração de DNS não será alterada indevidamente caso um novo servidor seja criado com base nessa AMI


5. Crie o uma tarefa no Agendador de Tarefa do Windows para executar o script toda vez que o servidor ligar
    - Abra o Task Scheduler
    - Action => Create Task...
    - Selecione Run Whether user is logged on or not
    - Digite o Nome da Task
    - Na aba Triggers => New => Em Begin the Task selecione *At Startup* => Ok
    - Na aba Actions => New => Em Program/script adicione o caminho do PowerShell "*C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe*" => Em Add Arguments adicine o caminho do Script "*-Command C:\scripts\route53-set-public-ip.ps1 -Noninteractive*" => OK
    - OK, Salve a task


6. Crie a Entrada DNS, caso não exista e teste
    - A entrada DNS deverá existir no Route53
    - Rode o Script e verifique se a alteração foi realizada corretamente
    - Desligue o Servidor, e Ligue novamente, então valide no Route53 que o endereço de IP foi alterado conforme ElasticIP alocado a instância




Referência: [Burnham.io](https://www.burnham.io/2017/02/dynamic-dns-using-amazon-route-53-and-powershell/#checking-our-public-ip)
