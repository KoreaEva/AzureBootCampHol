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

먼저 Queue를 추가할 Storage Accout를 먼저 생성한다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0023.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0024.png)<br>

Queue를 5개 생성한다. 

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0025.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0026.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0027.png)<br>

Queue에 값을 저장하기 위한 더미 앱을 생성한다. 이때 Windwos.Azure.Storage NuGet 패키지를 설치해준다. 
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0028.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0029.png)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0030.png)<br>

프로젝트가 생성되고 나면 우선 반복으로 Timer 객체를 이용해서 반복 동작할 수 있는 구조를 만들어야 합니다. 먼저 타이머를 사용할 수 있게 program.cs 파일에 namespace를 추가해야 합니다.

```csharp
using System.Timers;
```

타이머 객체를 멤버 변수로 정의 합니다.

```csharp
private static System.Timers.Timer SensorTimer;
```

이제 타이머를 설정할 수 있는 SetTimer( )를 작성합니다.

```csharp
  private static void SetTimer()
  {
        SensorTimer = new Timer(2000);
        SensorTimer.Elapsed += SensorTimer_Elapsed;
        SensorTimer.AutoReset = true;
        SensorTimer.Enabled = true;
  } 
```
타이머 객체에서 Interval을 2000으로 잡았기 때문에 이 타이머는 2초에 한번씩 이벤트가 발생 됩니다.. 발생된 이벤트를 처리하는 이벤트 핸들러를 작성하는데 이벤트 핸들러의 이름은 SensorTimer_Elapsed() 입니다.

SensorTimer_Elapsed 이벤트 핸들러를 작성합니다.

```csharp
    private async static void SensorTimer_Elapsed(object sender, ElapsedEventArgs e)
    {
        Console.WriteLine("The Elapsed event was raised at {0:HH:mm:ss.fff}", e.SignalTime);
    }
```

2초에 한번씩 이 이벤트 핸들러는 호출된다. 호출될 때마다 콘솔에서는 현재 이벤트 발생시간과 함께 메시지를 출력합니다.

Main() 안에서 SetTimer()는 코드와 함께 Enter 키를 누르면 프로그램이 멈출 수 있게 코드를 작성 합니다.

```csharp
    SetTimer();

    Console.WriteLine("\nPress the Enter key to exit the application...\n");
    Console.WriteLine("The application started at {0:HH:mm:ss.fff}", DateTime.Now);
    Console.ReadLine();
    SensorTimer.Stop();
    SensorTimer.Dispose();
```

여기까지 잘 작성되었으면 F5을 눌러서 실행해 본다. 그럼 2초에 한번씩 시간이 찍혀 나오는 모습을 볼 수 있습니다.

![https://github.com/KoreaEva/IoT/raw/master/Labs/IoT_Hub/images/device002.PNG](https://github.com/KoreaEva/IoT/raw/master/Labs/IoT_Hub/images/device002.PNG)<br>

이제 Queue와 통신을 하기 위한 부분을 추가한다. 먼저 관련 네임스페이스를 먼저 추가한다. 

```csharp
using Microsoft.Azure; // Namespace for CloudConfigurationManager
using Microsoft.WindowsAzure.Storage; // Namespace for CloudStorageAccount
using Microsoft.WindowsAzure.Storage.Queue; // Namespace for Queue storage types
```

Queue와 연결 할 수 있는 객체들을 생성해서 맴버 변수로 추가한다. 
```csharp
        private static CloudStorageAccount storageAccount = CloudStorageAccount.Parse("DefaultEndpointsProtocol=https;AccountName=gabdatastorage;AccountKey=R1LDolBsuYdQA0x28/Fd0QkFqT9v/ACzkBO8nPxbQzR0DL51Wn5iCuj2sNJ+3qrnV8LtUTCsyqbFO8hEsw7Iiw==;EndpointSuffix=core.windows.net");
        // Create the queue client.
        private static CloudQueueClient queueClient = storageAccount.CreateCloudQueueClient();

        // Retrieve a reference to a queue.
        private static CloudQueue queue1 = queueClient.GetQueueReference("queue1");
        private static CloudQueue queue2 = queueClient.GetQueueReference("queue2");
        private static CloudQueue queue3 = queueClient.GetQueueReference("queue3");
        private static CloudQueue queue4 = queueClient.GetQueueReference("queue4");
        private static CloudQueue queue5 = queueClient.GetQueueReference("queue5");

        private static DummySensor Sensor = new DummySensor();
```

실제 데이터를 가지고 실습을 하지 못하는 경우 가상의 센서를 만들어서 임의의 값이 계속 출력되는 방식으로 실습할 수 있다. 여기서는 더미 센서를 만들어 보기 이전에 먼제 데이터 모델을 먼저 만들어야 했다. 프로젝트에서 마우스 오른쪽 버튼을 클릭해서 Add -> Class를 차레로 선택한 다음 Class의 이름을 WetherModel 이라고 입력한다. 그리고 아래와 같이 데이터 모델을 작성한다.

```csharp
    public class WetherModel
    {
        public string DeviceID { get; set; }
        public int Temperature { get; set; }
        public int Humidity { get; set; }
        public int Dust { get; set; }
    }
```

같은 방식으로 Class를 생성한 다음 Class의 이름을 DummySensor라고 입력하고 아래와 같이 더미 센서를 작성한다.

```csharp
    class DummySensor
    {
        private Random _Random = new Random();

        public WetherModel GetWetherData(string deviceID)
        {
            WetherModel wetherModel = new WetherModel();

            wetherModel.DeviceID = deviceID;
            wetherModel.Temperature = _Random.Next(25, 32);
            wetherModel.Humidity = _Random.Next(60, 80);
            wetherModel.Dust = 50 + wetherModel.Temperature + _Random.Next(1, 5);
            return wetherModel;
        }
    }
```

더미 센서의 GetWetherData()를 호출할 때 마다 난수를 통해서 임의의 센서값을 생성해서 돌려주는 부분이 완성되었다. 나중에 머신러닝 실습 등에 이용할 수 있게 온도와 먼지는 상관관계를 만들어 두었다.마지막으로 데이터를 송신하는 부분을 다음과 같이 수정한다. 

```csharp
        private async static void SensorTimer_Elapsed(object sender, ElapsedEventArgs e)
        {
            string json = Newtonsoft.Json.JsonConvert.SerializeObject(Sensor.GetWetherData("Device1"));

            Console.WriteLine(json);

            CloudQueueMessage message = new CloudQueueMessage(json);

            //Queue에 분산해서 넣는 부분
            Random random = new Random();

            int number = random.Next(5);

            switch (number)
            {
                case 1:
                    {
                        queue1.AddMessage(message);
                        Console.WriteLine("Queue 1 {0:HH:mm:ss.fff}", e.SignalTime);
                        break;
                    }
                case 2:
                    {
                        queue2.AddMessage(message);
                        Console.WriteLine("Queue 2 {0:HH:mm:ss.fff}", e.SignalTime);
                        break;
                    }
                case 3:
                    {
                        queue3.AddMessage(message);
                        Console.WriteLine("Queue 3 {0:HH:mm:ss.fff}", e.SignalTime);
                        break;
                    }
                case 4:
                    {
                        queue4.AddMessage(message);
                        Console.WriteLine("Queue 4 {0:HH:mm:ss.fff}", e.SignalTime);
                        break;
                    }
                case 5:
                    {
                        queue5.AddMessage(message);
                        Console.WriteLine("Queue 5 {0:HH:mm:ss.fff}", e.SignalTime);
                        break;
                    }
            }
        }
```

다시 Azure Functions 프로젝트에서 새로운 Azure 함수를 추가하는 데 이번에는 Queue Trigger를 추가한다. 
이때 설정 파일에 연결 문자열은 별도로 설정해야 한다. 

```csharp
using System;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Host;

using Microsoft.Azure.Documents;
using Microsoft.Azure.Documents.Client;
using Newtonsoft.Json.Linq;

namespace GabFunc
{
    public static class QueueTrigger2
    {
        private static string CosmosWifiEndpointUrl = Environment.GetEnvironmentVariable("CosmosEndpoint");
        private static string CosmosPrimaryKey = Environment.GetEnvironmentVariable("CosmosPrimaryKey");
        private static DocumentClient client = new DocumentClient(new Uri(CosmosWifiEndpointUrl), CosmosPrimaryKey);
        private static string DatabaseName = Environment.GetEnvironmentVariable("CosmosDatabase");
        private static string DataCollectionName = Environment.GetEnvironmentVariable("CosmosCollection");

        [FunctionName("QueueTrigger2")]
        public async static void Run([QueueTrigger("queue2", Connection = "QueueConnectionString")]string myQueueItem, TraceWriter log)
        {
            log.Info($"C# Queue---2 trigger function processed: {myQueueItem}");

            //전달받은 JSON을 기반으로 HttpModel 클래스를 만들고 고유 아이디를 첨부하는 부분
            JObject jobjct = JObject.Parse(myQueueItem);
            SensorModel model = new SensorModel();
            model.id = generateID();
            model.jobject = jobjct;

            //CosmosDB에 입력하는 코드
            await client.CreateDocumentAsync(UriFactory.CreateDocumentCollectionUri(DatabaseName, DataCollectionName), model);

        }

        /// 문서의 고유한 ID를 생성하는 함수
        public static string generateID()
        {
            return string.Format("{0}_{1:N}", System.DateTime.Now.Ticks, Guid.NewGuid());
        }
    }
}
```

같은 방법으로 Queue마다 하나씩 Queue Trigger를 추가해 놓고 결과를 확인한다. 

## 3. EventHub의 사용

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/a3.PNG)<br>

EventHub는 HTTP 혹은 AMQP로 오는 요청을 처리할 수 서비스로 폭주하는 서비스를 감당할 수 있게 잘 설계 되어 있을 뿐만 아니라 자체적인 Partition을 가지고 있어서 데이터를 손실 없이 두 단의 작업으로 연계 할 수 있다. 
 여기서는 EventHub에 데이터가 들어 올 때 마다 EventHub Trigger를 사용해서 CosmosDB에 저장하는 코드를 작성한다. 

### 4. EventHub를 통해 들어온 데이터의 관리

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/a4.PNG)<br>

EventHub를 통해서 수집된 데이터를 백업하고 일정 시간이 지난 데이터를 삭제하는 등 데이터를 관리하는 방법을 살펴 본다. 