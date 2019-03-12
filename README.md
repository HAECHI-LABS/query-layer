# HAECHI QUERY LAYER

## Intro
블록체인에서 발생하는 많은 데이터를 쉽게 가공하여 원하는 형태로 가져옵니다.

블록체인에서 발생하는 다양한 형태의 데이터를 빠르고 쉽게 원하는 형태로 조회하기 어렵습니다. 예를 들어 ERC 토큰 컨트랙트에서 1000 토큰 이상 보유하고 있는 계좌를 조회하고 싶은경우 혹은 발생한 transfer 이벤트중 특정 계좌로 전송된 모든 이벤트를 검색하고 싶은 경우에, 기존 이더리움 컨트랙트에서는 조건문을 이용한 조회가 매우 불편할뿐만 아니라 인덱싱이 되어있지 않아서 빠르게 데이터를 조회하기 어렵습니다.

HAECHI Query Layer는 바로 이러한 불편함을 해소하기 위한 미들웨어 솔루션 입니다. 원하는 이벤트를 자동으로 수집하고 원하는 형태로 가공하여 인덱싱을 하기 때문에, 블록체인 서비스 개발 및 운영에 필요한 이벤트를 쉽고 빠르게 원하는 형태로 조회할 수 있습니다. HAECHI Query Layer는 Aws Lambda와 같이 event process 로직을 함수 단위로 쉽게 배포하여 등록할 수 있습니다.

## Architecture

![](/img/arch.png)

### overall flow

1. 구독하고 싶은 컨트랙트가 있다면, 사용자는 *Source*를 배포합니다. *Soruce*는 블록체인 정보를 실시간으로 fetch합니다. 

2. *Soruce*로 부터 구독하고 있는 이벤트 정보를 상황에 맞게 가공하고 싶다면, 사용자는 *Function*을 배포합니다. *Function*은 사용자가 지정한 이벤트가 발생할 경우 실행됩니다. Function에서 처리된 이벤트 정보는 *Database*에 저장됩니다.
3. *QueryService*는, *Function*에서 가공된 정보를 쉽게 불러올 수 있는 endpoint interface입니다. GraphQL, RESTful API등으로 클라이언트에서 서버 없이도 쉽게 *Database*에 있는 정보를 불러올 수 있습니다.



## Components
### Source
블록체인의 데이터를 가져오는 component입니다.  HAECHI Query Layer에서 제공하는 데이터는 크게 두 가지로 나뉩니다. 

블록정보
이더리움 블록에 담긴 메타 정보들을 제공합니다.
block number
transaction hash

컨트랙트 이벤트
특정 컨트랙트 주소에서 발생한 이벤트 정보를 제공합니다.
event name
contract address
event properties

### Function
Source로 부터 발생하는 이벤트를 가공하여 Database에 저장하는 사용자 정의 함수입니다. 함수는 다음과 같은 Input을 요구합니다
source code
dependency file( e.g. package.json )
topic name(e.g. “userid.contractAddress.event”)

Function code의 예시는 다음과 같습니다.

```typescript
@Handler()
exports.function = function(event: IDepositStakeEvent, db: IStore) {
    
    // find depositor
    let depositor: Depositor = (await db.findById(event.depositor) as Depositor);
    
    // if depositor exists
    if (depositor){
      const sender = depositor.senders.find(s => s.id == event.sender);
      
      // if sender exists increase amount
      if (sender){
        sender.amount = Number(sender.amount) + Number(event.amount);
      }else{
        // if sender does not exists, create new sender
        depositor.senders.push(new Sender(event.sender, event.amount));
      }
    // if not exists
    }else{
      // create new depositor
      depositor = new Depositor(event.depositor, [new Sender(event.sender, event.amount)]);
    }
    
    await db.save(depositor.id, depositor);
  }
}
```
위의 예제는 IDepositStakeEvent 의 depositer로 deposit을 한 모든 sender와 deposit양을 list 형태로 DB에 저장하는 예제 입니다. 위의 예제처럼 function단위로 쉽게 event processor를 등록하여 event를 핸들링 할 수 있습니다.

### Query Service

Query Service는 Function에서 가공하여 저장한 이벤트 정보를 쉽게 조회할 수 있도록 endpoint를 제공합니다. 기본적으로 graphql endpoint가 제공되며 DB에 저장한 Entity들을 조회할 수 있는 findById, findAll, findByProperty, findFirst, findLast의 api가 자동생성 됩니다. 
