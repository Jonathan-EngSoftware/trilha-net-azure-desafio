# Guia de Implantação no Microsoft Azure

## Visão Geral
Esta aplicação está configurada para ser implantada no Microsoft Azure utilizando:
- **App Service** para hospedar a API
- **SQL Database** para o banco de dados relacional
- **Azure Table Storage** para armazenar os logs de funcionários

## Pré-requisitos
1. Conta ativa no Microsoft Azure
2. Azure CLI instalado
3. Visual Studio ou Visual Studio Code com extensões do Azure

## Passos para Implantação

### 1. Criar Recursos no Azure

#### 1.1 Criar Resource Group
```bash
az group create --name rg-trilha-net-azure --location "Brazil South"
```

#### 1.2 Criar SQL Database
```bash
# Criar SQL Server
az sql server create \
  --name trilha-net-sql-server \
  --resource-group rg-trilha-net-azure \
  --location "Brazil South" \
  --admin-user sqladmin \
  --admin-password "SuaSenhaSegura123!"

# Criar SQL Database
az sql db create \
  --resource-group rg-trilha-net-azure \
  --server trilha-net-sql-server \
  --name trilha-net-database \
  --service-objective Basic
```

#### 1.3 Criar Storage Account para Azure Table
```bash
az storage account create \
  --name trilhanetazurestorage \
  --resource-group rg-trilha-net-azure \
  --location "Brazil South" \
  --sku Standard_LRS
```

#### 1.4 Criar App Service
```bash
# Criar App Service Plan
az appservice plan create \
  --name trilha-net-plan \
  --resource-group rg-trilha-net-azure \
  --sku B1 \
  --is-linux

# Criar Web App
az webapp create \
  --resource-group rg-trilha-net-azure \
  --plan trilha-net-plan \
  --name trilha-net-azure-api \
  --runtime "DOTNETCORE:8.0"
```

### 2. Configurar Strings de Conexão

#### 2.1 Obter Connection Strings
```bash
# SQL Database Connection String
az sql db show-connection-string \
  --client ado.net \
  --name trilha-net-database \
  --server trilha-net-sql-server

# Storage Account Connection String
az storage account show-connection-string \
  --name trilhanetazurestorage \
  --resource-group rg-trilha-net-azure
```

#### 2.2 Configurar no App Service
```bash
# Configurar SQL Connection String
az webapp config connection-string set \
  --resource-group rg-trilha-net-azure \
  --name trilha-net-azure-api \
  --connection-string-type SQLAzure \
  --settings ConexaoPadrao="Server=tcp:trilha-net-sql-server.database.windows.net,1433;Initial Catalog=trilha-net-database;Persist Security Info=False;User ID=sqladmin;Password=SuaSenhaSegura123!;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"

# Configurar Storage Connection String
az webapp config appsettings set \
  --resource-group rg-trilha-net-azure \
  --name trilha-net-azure-api \
  --settings SAConnectionString="DefaultEndpointsProtocol=https;AccountName=trilhanetazurestorage;AccountKey=SUA_ACCOUNT_KEY;EndpointSuffix=core.windows.net" \
  AzureTableName="FuncionarioLog"
```

### 3. Executar Migrations

#### 3.1 Configurar Firewall do SQL Server
```bash
# Permitir acesso do Azure
az sql server firewall-rule create \
  --resource-group rg-trilha-net-azure \
  --server trilha-net-sql-server \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Permitir seu IP (substitua pelo seu IP público)
az sql server firewall-rule create \
  --resource-group rg-trilha-net-azure \
  --server trilha-net-sql-server \
  --name AllowMyIP \
  --start-ip-address SEU_IP \
  --end-ip-address SEU_IP
```

#### 3.2 Executar Migrations
```bash
# Atualizar connection string no appsettings.json temporariamente
# Executar migrations
dotnet ef database update

# Ou usar o comando direto com connection string
dotnet ef database update --connection "Server=tcp:trilha-net-sql-server.database.windows.net,1433;Initial Catalog=trilha-net-database;Persist Security Info=False;User ID=sqladmin;Password=SuaSenhaSegura123!;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
```

### 4. Deploy da Aplicação

#### 4.1 Via Azure CLI
```bash
# Fazer deploy do código
az webapp deployment source config-zip \
  --resource-group rg-trilha-net-azure \
  --name trilha-net-azure-api \
  --src publish.zip
```

#### 4.2 Via Visual Studio
1. Clique com botão direito no projeto
2. Selecione "Publish"
3. Escolha "Azure"
4. Selecione "Azure App Service (Windows)"
5. Faça login na sua conta Azure
6. Selecione o App Service criado
7. Clique em "Publish"

### 5. Verificações Pós-Deploy

#### 5.1 Testar a API
- Acesse: `https://trilha-net-azure-api.azurewebsites.net`
- Verifique o Swagger: `https://trilha-net-azure-api.azurewebsites.net/swagger`

#### 5.2 Verificar Logs
```bash
# Ver logs do App Service
az webapp log tail \
  --resource-group rg-trilha-net-azure \
  --name trilha-net-azure-api
```

## Funcionalidades Implementadas

### Sistema de Logs
- ✅ Todos os CRUDs de funcionário geram logs automáticos
- ✅ Logs são armazenados no Azure Table Storage
- ✅ Logs incluem: tipo de ação, dados do funcionário em JSON, timestamp

### Endpoints Disponíveis
- `GET /Funcionario/{id}` - Obter funcionário por ID
- `POST /Funcionario` - Criar novo funcionário
- `PUT /Funcionario/{id}` - Atualizar funcionário
- `DELETE /Funcionario/{id}` - Deletar funcionário

### Estrutura dos Logs
Cada operação CRUD gera um log com:
- **PartitionKey**: Departamento do funcionário
- **RowKey**: GUID único
- **TipoAcao**: Inclusao, Atualizacao, ou Remocao
- **JSON**: Dados completos do funcionário
- **Timestamp**: Data/hora da operação

## Monitoramento e Manutenção

### Application Insights (Recomendado)
```bash
# Criar Application Insights
az monitor app-insights component create \
  --app trilha-net-insights \
  --location "Brazil South" \
  --resource-group rg-trilha-net-azure

# Configurar no App Service
az webapp config appsettings set \
  --resource-group rg-trilha-net-azure \
  --name trilha-net-azure-api \
  --settings APPINSIGHTS_INSTRUMENTATIONKEY="SUA_INSTRUMENTATION_KEY"
```

### Backup e Recuperação
- SQL Database: Backup automático habilitado
- Storage Account: Configurar replicação geográfica se necessário

## Custos Estimados (Região Brazil South)
- App Service (B1): ~R$ 50/mês
- SQL Database (Basic): ~R$ 25/mês  
- Storage Account: ~R$ 5/mês
- **Total estimado**: ~R$ 80/mês

## Segurança
- ✅ Conexões criptografadas (HTTPS/TLS)
- ✅ Firewall configurado no SQL Server
- ✅ Autenticação via Azure AD (recomendado para produção)
- ✅ Secrets gerenciados via App Settings

## Troubleshooting
1. **Erro de conexão SQL**: Verificar firewall rules
2. **Erro 500**: Verificar logs do App Service
3. **Tabela não encontrada**: Executar migrations
4. **Storage não acessível**: Verificar connection string

Para mais informações, consulte a documentação oficial do Azure.