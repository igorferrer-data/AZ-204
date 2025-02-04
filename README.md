# AZ-204
Criando um microsserviço servless para validação de CPF

## ⚙️ Configuração do Ambiente Local

Instalar as dependências:
- Visual Studio Code (ou Visual Studio)
- Azure Functions Core Tools
- .NET SDK 8
- Azure CLI

## 🚀 Criação da Azure Function

Execute os seguintes comandos no terminal:

```bash
mkdir cpf-validation-function
cd cpf-validation-function
```

### Inicializar o projeto com .NET

```bash
func init --worker-runtime dotnet --target-framework net8.0
```

### Criar a função com trigger HTTP

```bash
func new --template "HttpTrigger" --name ValidateCpf
```

## ✍️ Implementação do Código

Abra `ValidateCpf.cs` e substitua o conteúdo pelo seguinte código, que inclui a validação de CPF:

```bash
using System;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;

namespace CpfValidationFunctionApp
{
    public static class ValidateCpf
    {
        [FunctionName("ValidateCpf")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req,
            ILogger log)
        {
            log.LogInformation("Iniciando a validação de CPF.");

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);
            string cpf = data?.cpf;

            if (string.IsNullOrEmpty(cpf))
            {
                return new BadRequestObjectResult("Por favor, informe um CPF.");
            }

            bool isValid = ValidateCpfNumber(cpf);
            return isValid
                ? new OkObjectResult($"CPF {cpf} é válido.")
                : new BadRequestObjectResult($"CPF {cpf} é inválido.");
        }

        private static bool ValidateCpfNumber(string cpf)
        {
            cpf = new string(cpf.Where(char.IsDigit).ToArray());

            if (cpf.Length != 11 || cpf.All(c => c == cpf[0])) return false;

            int[] multiplicadores1 = { 10, 9, 8, 7, 6, 5, 4, 3, 2 };
            int[] multiplicadores2 = { 11, 10, 9, 8, 7, 6, 5, 4, 3, 2 };

            string tempCpf = cpf.Substring(0, 9);
            int soma = tempCpf.Select((t, i) => int.Parse(t.ToString()) * multiplicadores1[i]).Sum();
            int resto = (soma * 10) % 11;
            if (resto == 10 || resto == 11) resto = 0;
            if (resto != int.Parse(cpf[9].ToString())) return false;

            tempCpf += cpf[9];
            soma = tempCpf.Select((t, i) => int.Parse(t.ToString()) * multiplicadores2[i]).Sum();
            resto = (soma * 10) % 11;
            if (resto == 10 || resto == 11) resto = 0;

            return resto == int.Parse(cpf[10].ToString());
        }
    }
}
```

##⚡ Testando a Função Localmente

Inicie a Azure Function localmente:

```bash
func start
```

Abra o Postman e envie uma requisição POST para `http://localhost:7071/api/ValidateCpf`, com um JSON:

```json
{
  "cpf": "12345678909"
}
```

## 🌐 Publicação no Azure

Crie o recurso da Azure Function:

```bash
az login
az functionapp create --name "validatecpfapp" --resource-group "myResourceGroup" --consumption-plan-location "EastUS" --runtime dotnet --runtime-version 8 --functions-version 4
```

Faça o deploy:

```bash
func azure functionapp publish validatecpfapp
```

## 🔑 Chaves de Autenticação

Para passar a chave de acesso na requisição:

1. Vá para o Azure Portal > Function App > Chaves de Função.
2. Copie a chave e use no Postman como parâmetro `code`.
