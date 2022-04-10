# Low-level API
스파크에는 두 종류의 저수준 API가 있습니다.
하나는 분산 데이터 처리를 위한 RDD이며, 다른 하나는 분산형 공유 변수를 배포하고 다루기 위한 API입니다.
스파크의 모든 워크로드는 저수준 기능을 사용하는 형태로 컴파일되므로 이를 이해하는 것은 많은 도움이 될 수 있습니다.
SparkContext는 저수준 API 기능을 사용하기 위한 진입 지점이며, SparkSession을 이용해 SparkContext에 접근할 수 있습니다.

low level API를 사용하는 예시입니다.
●    고수준 API에서 제공하지 않는 기능이 필요한 경우.
●    RDD를 사용해 개발된 기존 코드를 유지해야 하는 경우
●    사용자가 정의한 공유 변수를 다뤄야 하는 경우.

## RDD
사용자는 두 가지 타입의 RDD를 만들 수 있습니다.
하나는 '제네릭 RDD' 타입이고 다른 하나는 키 기반의 집계가 가능한 '키-값' RDD입니다.
둘 다 객체의 컬렉션을 표현하지만 '키-값' RDD는 특수 연산뿐만 아니라 키를 이용한 사용자 지정 '파티셔닝' 개념을 가지고 있습니다.

## Repartition과 Coalesce
파티셔닝 스키마와 파티션 수를 포함해 클러스터 전반의 물리적 데이터 구성을 제어할 수 있습니다.
'repartition' 메서드는 파티션 수를 늘리거나 줄일 수 있지만, 처리 시 무조건 전체 데이터를 셔플합니다.
파티션 수를 늘리면 맵 타입이나 필터 타입의 연산을 수행할 때 병렬 처리 수준을 높일 수 있습니다.
'coalesce' 메서드는 파티션을 재분배할 때 발생하는 데이터 셔플을 방지하기 위해 동일한 워커에 존재하는 파티션을 합치는 메서드입니다.

## Custom partitioning
사용자 정의 파티셔닝은 RDD를 사용하는 가장 큰 이유 중 하나입니다.
파티셔닝은 데이터 치우침 같은 문제를 피하고자 클러스터 전체에 걸쳐 데이터를 균등하게 분배하는 겁니다.

파티션 함수는 보통 사용자 지정 Partitioner를 의미합니다.
구조적 API에서는 사용자 정의 파티셔너를 파라미터로 사용할 수 없습니다.

사용자 정의 파티셔너를 사용하려면 구조적 API로 RDD를 얻고 사용자 정의 파티셔너를 적용한 다음 다시 DataFrame이나 Dataset으로 변환해야 합니다.
이 방법은 필요시에만 사용자 정의 파티셔닝을 사용할 수 있으므로 구조적 API와 RDD의 장점을 모두 활용할 수 있습니다.

사용자 정의 파티셔닝을 사용하려면 Partitioner를 확장한 클래스를 구현해야 합니다.
문제에 대한 업무 지식을 충분히 가지고 있는 경우에만 사용해야 합니다.
단일 값이나 다수 값(다수 컬럼)을 파티셔닝해야 한다면 DataFrame API를 사용하는 것이 좋습니다.

```scala
  val df = spark.read.option("header", "true").option("inferSchema", "true")
    .csv("/data/retail-data/all/")
  val rdd = df.coalesce(10).rdd
```
```python
  df = spark.read.option("header", "true").option("inferSchema", "true")\
    .csv("/data/retail-data/all/")
  rdd = df.coalesce(10).rdd
```
HashPartitioner와 RangePartitioner는 RDD API에서 사용할 수 있는 내장형 파티셔너로 매우 기초적인 기능을 제공합니다.
각각 이산형과 연속형 값을 다룰 때 사용합니다.
매우 큰 데이터나 심각하게 치우친 키를 다뤄야 한다면 고급 파티셔닝 기능을 사용해야 합니다.

## Dataset과 RDD + Case Class
Dataset은 구조적 API가 제공하는 풍부한 기능과 최적화 기법을 제공한다는 것이 가장 큰 차이점입니다.
예를 들면, Dataset을 사용하면 JVM 데이터 타입과 스파크 데이터 타입 중 어떤 것을 쓸지 고민하지 않아도 됩니다.
어떤 것을 사용하더라도 성능은 동일하므로 가장 쉽게 사용할 수 있고 유연하게 대응할 수 있는 데이터 타입을 선택하면 됩니다.

## Kryo serialization
Kryo는 자바 직렬화보다 약 10배 이상 성능이 좋으며 더 간결합니다.
사용할 클래스를 사전에 등록해야 합니다.

spark.serializer 설정으로 워커 노드 간 데이터 셔플링과 RDD를 직렬화해 디스크에 저장하는 용도로 사용할 시리얼라이저를 지정할 수 있습니다.
SparkConf를 사용해 잡을 초기화하는 시점에서 spark.serializer 속성값을 org.apache.spark.serializer.KryoSerializer로 설정해 Kryo를 사용할 수 있습니다.
Kryo가 기본 값이 아닌 유일한 이유는 사용자가 직접 클래스를 등록해야 하기 때문입니다.
Kryo에 사용자 정의 클래스를 등록하려면 registerKryoClasses 메서드를 사용합니다.

## Distributed shared variable
분산형 공유 변수에는 브로드캐스트 변수와 어큐뮬레이터라는 두 개의 타입이 존재합니다.
클러스터에서 실행할 때 특별한 속성을 가진 사용자 정의 함수에서 이 변수를 사용할 수 있습니다.
특히 어큐뮬레이터를 사용하면 모든 태스크의 데이터를 공유 결과에 추가할 수 있습니다.
반면 브로드캐스트 변수를 사용하면 모든 워커 노드에 큰 값을 저장하므로 재전송 없이 많은 스파크 액션에서 재사용할 수 있습니다.

## Broadcast
브로드캐스트 변수는 변하지 않는 값(불변성 값)을 클러스터에서 효율적으로 공유하는 방법을 제공합니다.
태스크에서 드라이버 노드의 변수를 사용할 때는 클로저 함수 내부에서 단순하게 참조하는 방법을 사용하지만 비효율적입니다.
그 이유는 클로저 함수에서 변수를 사용할 때 워커 노드에서 여러 번(태스크당 한 번) 역직렬화가 일어나기 때문입니다.
게다가 여러 스파크 액션과 잡에서 동일한 변수를 사용하면 잡을 실행할 때마다 워커로 큰 변수를 재전송합니다.
브로드캐스트 변수는 모든 태스크마다 직렬화하지 않고 클러스터의 모든 머신에 캐시하는 불변성 공유 변수입니다.

broadcast 변수의 value 메서드를 사용해서 값을 참조할 수 있습니다.
```scala
  val supplementalData = Map("Spark" -> 1000)
  val suppBroadcast = spark.sparkContext.broadcast(supplementalData)
  suppBroadcast.value
```
```python
  supplementalData = {"Spark":1000}
  suppBroadcast = spark.sparkContext.broadcast(supplementalData)
  suppBroadcast.value
```
## Accumulator
어큐뮬레이터는 트랜스포메이션 내부의 다양한 값을 갱신하는 데 사용합니다.
어큐뮬레이터는 스파크 클러스터에서 로우 단위로 안전하게 값을 갱신할 수 있는 변경 가능한 변수를 제공합니다.
그리고 디버깅용이나 저수준 집계 생성용으로 사용할 수 있습니다.

어큐뮬레이터의 값은 액션을 처리하는 과정에서만 갱신됩니다.
스파크는 각 태스크에서 어큐뮬레이터를 한 번만 갱신하도록 제어합니다.
따라서 재시작한 태스크는 어큐뮬레이터값을 갱신할 수 없습니다.