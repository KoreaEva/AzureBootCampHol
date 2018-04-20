# AzureBootCampHol
본 자료는 Globa Azure Bootcamp 2018 행사중에서 PaaS Hands on Lab을 위한 자료입니다. 이 자료는 PaaS 기반으로 서비스를 구성하고 어떻게 하면 폭주하는 트래픽을 처리 할 수 있을까를 고민하는 세션입니다. 
 이 세션에서는 1초에 약 5000건의 요청이 쏟아질 때 어떻게 이 요청을 소화 할 수 있을 것인가를 함께 고민해 봅니다. 

## 필수 소프트웨어

Visual Studio 2017 Community[https://www.visualstudio.com/ko/](https://www.visualstudio.com/ko/)<br>
Postman [https://www.getpostman.com/apps](https://www.getpostman.com/apps)<br>
Storage Explorer [https://azure.microsoft.com/en-us/features/storage-explorer/](https://azure.microsoft.com/en-us/features/storage-explorer/)<br>

## 1. Azure Function을 사용한 데이터의 처리
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/a1.PNG)<br>

 먼저 Azure Functions은 가장 Serverless로 아키텍처를 구성할 때 가장 먼저 떠오르는 서비스입니다. Azure Functions와 CosmosDB를 생성했을 때 우리는 트래픽에 대한 예측을 할 수 있습니다. 

먼저 리소스 그룹을 만든다.

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0001.png)<br>

리소스 그룹이 완성되면 Azure Functions를 추가한다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0002.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0003.png)<br>

Visual Studio에서 새로운 Azure Functions 프로젝트를 추가한다. 이때 .NET Framework 버전을 선택하고 Azure Functions를 만들때 추가 되어 있는 Storage를 선택한다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0006.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0008.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0007.png)<br>

새 Azure 함수를 추가한다. 이때 Trigger는 Http Trigger를 선택한다. 그리고 접근 권한은 Anonymous를 선택한다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0004.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0005.png)<br>

새로운 함수가 추가되면 바로 추가 되어 있는 샘플 코드를 실행한다. 그리고 출력되는 주소로 Postman을 이용해서 접속해서 테스트 한다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0009.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0010.png)<br>

이제 데이터를 저장할 Cosmos DB를 생성한다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0011.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0012.png)<br>

CosmosDB가 잘 추가 되었다면 새로운 컬랙션을 추가한다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0013.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0014.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0015.png)<br>

다시 Visual Studio로 돌아와서 NuGet 패키지를 추가하는데 Microsoft.Azure.DocumentDB를 찾아서 추가한다.

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0016.png)<br>

필요한 네임스페이스를 추가한다. 

```csharp
using System;
using Microsoft.Azure.Documents;
using Microsoft.Azure.Documents.Client;
using Newtonsoft.Json.Linq;
```

전역 변수를 선언하는데 local.settings.json의 내용을 가져올 수 있게 맴버 변수로 추가한다. 

```csharp
        static private string CosmosWifiEndpointUrl = Environment.GetEnvironmentVariable("CosmosEndpoint");
        static private string CosmosPrimaryKey = Environment.GetEnvironmentVariable("CosmosPrimaryKey");
        private static DocumentClient client = new DocumentClient(new Uri(CosmosWifiEndpointUrl), CosmosPrimaryKey);
        private static string DatabaseName = Environment.GetEnvironmentVariable("CosmosDatabase");
        private static string DataCollectionName = Environment.GetEnvironmentVariable("CosmosCollection");
```

Run()과 generateID() 메소드를 추가한다. 

```csharp
        [FunctionName("HttpTrigger")]
        public static async Task<HttpResponseMessage> Run([HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)]HttpRequestMessage req, TraceWriter log)
        {
            log.Info("C# HTTP trigger function processed a request.");

            //Cosmos DB Database와 Collection이 없으면 생성하는 부분 추후 서비스가 안정되면 삭제될 코드
            await client.CreateDatabaseIfNotExistsAsync(new Database { Id = DatabaseName });
            await client.CreateDocumentCollectionIfNotExistsAsync(UriFactory.CreateDatabaseUri(DatabaseName), new DocumentCollection { Id = DataCollectionName });

            // parse query parameter
            string message = req.GetQueryNameValuePairs()
                .FirstOrDefault(q => string.Compare(q.Key, "message", true) == 0)
                .Value;

            if (message == null)
            {
                // Get request body
                dynamic data = await req.Content.ReadAsAsync<object>();
                message = data?.name;
            }

            if (message != null || message != "")
            {
                //전달받은 JSON을 기반으로 HttpModel 클래스를 만들고 고유 아이디를 첨부하는 부분
                JObject jobjct = JObject.Parse(message);
                SensorModel model = new SensorModel();
                model.id = generateID();
                model.jobject = jobjct;

                //CosmosDB에 입력하는 코드
                await client.CreateDocumentAsync(UriFactory.CreateDocumentCollectionUri(DatabaseName, DataCollectionName), model);
            }

            return message == null
                ? req.CreateResponse(HttpStatusCode.BadRequest, "Please pass a name on the query string or in the request body")
                : req.CreateResponse(HttpStatusCode.OK, "Hello " + message);
        }

        /// 문서의 고유한 ID를 생성하는 함수
        public static string generateID()
        {
            return string.Format("{0}_{1:N}", System.DateTime.Now.Ticks, Guid.NewGuid());
        }
```

필요한 Entity class를 추가한다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0017.png)<br>

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

using Newtonsoft.Json.Linq;

namespace GabFunc
{
    class SensorModel
    {
        public string id { get; set; }
        public JObject jobject { get; set; }
    }
}
```

마지막으로 필요한 설정값을 찾아서 local.settings.json에 추가한다. 
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "DefaultEndpointsProtocol=https;AccountName=gabfuncstorage;AccountKey=B4yaYjrWkEYrssM8xLZswaUZDG1hEYAQI8065fue7qvLmxu/pKS+gSqm0X0OFtYOZAb1QsRi7MPC23do9QEejg==",
    "AzureWebJobsDashboard": "DefaultEndpointsProtocol=https;AccountName=gabfuncstorage;AccountKey=B4yaYjrWkEYrssM8xLZswaUZDG1hEYAQI8065fue7qvLmxu/pKS+gSqm0X0OFtYOZAb1QsRi7MPC23do9QEejg==",
    "CosmosEndpoint": "https://gabcosmos.documents.azure.com:443/",
    "CosmosPrimaryKey": "zQgpoPKoueHG34lezYIvfcU8BvjyRbCOlfAmCFM27xnHBRRQLoDEpcuHbFd9wlEjTfKUJE0CBSKUxhpT08afeA==",
    "CosmosDatabase": "gabdatabase",
    "CosmosCollection": "sensordata"
  }
```

다시 포스트맨으로 접속해서 JSON 코드를 입력해 본다.

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0018.png)<br>

문제가 없으면 직접 배포해 본다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0019.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0020.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0021.png)<br>

배포가 끝나고 나면 Application setting 값을 마지막으로 옮겨준다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0022.png)<br>


## 2. Azure Storage에서 제공하는 Queue를 사용한 처리
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/a2.PNG)<br>

Azure Functions을 사용했을 때 단시간에 급격히 증가하는 트래픽을 어떻게 처리 할 것인가를 고민 할때 또 하나의 해결 방법은 Azure Functions 앞에 Queue를 두는 방법입니다. Queue는 최대 1초 2000개의 요청을 소화 할 수 있기 때문에 넉넉하게 10개의 Queue를 만들고 Random하게 Queue를 선택해서 보낸다면 충분히 트래픽을 소화 할 수 있을 것으로 기대 된다. 
 단 이때 Queue 개수가 많아지면 관리가 점점 어려워지는 문제가 있지만 비교적 저렴한 비용으로 충분한 성능을 얻을 수 있는 장점이 있다. 

## 3. EventHub의 사용

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/a3.PNG)<br>

EventHub는 HTTP 혹은 AMQP로 오는 요청을 처리할 수 서비스로 폭주하는 서비스를 감당할 수 있게 잘 설계 되어 있을 뿐만 아니라 자체적인 Partition을 가지고 있어서 데이터를 손실 없이 두 단의 작업으로 연계 할 수 있다. 
 여기서는 EventHub에 데이터가 들어 올 때 마다 EventHub Trigger를 사용해서 CosmosDB에 저장하는 코드를 작성한다. 

### 4. EventHub를 통해 들어온 데이터의 관리

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/a4.PNG)<br>

EventHub를 통해서 수집된 데이터를 백업하고 일정 시간이 지난 데이터를 삭제하는 등 데이터를 관리하는 방법을 살펴 본다. 