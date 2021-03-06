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

![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0001.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0002.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0003.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0004.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0005.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0006.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0007.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0008.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0009.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0010.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0011.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0012.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0013.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0014.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0015.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0016.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0017.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0018.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0019.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0020.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0021.PNG)<br>
![Azure Functions](https://github.com/KoreaEva/AzureBootCampHol/blob/master/images/01/0022.PNG)<br>


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