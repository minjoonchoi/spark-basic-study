# Considerations for testing and performance tuning

## Testing

## various input data
입력 데이터로 인해 발생할 수 있는 다양한 예외 상황을 테스트하는 코드를 작성해야 합니다.

## business logic
실제와 유사한 데이터를 사용해 비즈니스 로직을 꼼꼼하게 테스트해야 함을 의미합니다.
이 유형의 테스트에서는 '스파크 단위 테스트'를 작성하지 않도록 조심해야 합니다.

## output data and schema
결과 데이터가 스키마에 맞는 적절한 형태로 반환될 수 있도록 제어해야 합니다.

## SparkSession management
스파크 로컬 모드 덕분에 JUnit이나 ScalaTest 같은 단위 테스트용 프레임워크로 비교적 쉽게 스파크 코드를 테스트할 수 있습니다.
로컬 모드의 SparkSession을 만들어 사용하기만 하면 됩니다.
스파크 코드에서 의존성 주입 방식으로 SparkSession을 관리하도록 만들어야 합니다.
즉, SparkSession을 한 번만 초기화하고 런타임 환경에서 함수와 클래스에 전달하는 방식을 사용하면 테스트 중에 SparkSession을 쉽게 교체할 수 있습니다.

## API selection
어떤 팀이나 프로젝트에서는 개발 속도를 올리기 위해 덜 엄격한 SQL과 DataFrame API를 사용할 수 있고 다른 팀에서는 타입 안정성을 얻기 위해 Dataset과 RDD API를 사용할 수 있습니다.
RDD API는 파티셔닝 같은 저수준 API의 기능이 필요한 경우에만 사용합니다.
Dataset API를 사용하면 성능을 최적화할 수 있으며 앞으로도 더 많은 성능 최적화 방식을 제공할 가능성이 높습니다.
대규모 애플리케이션이나 저수준 API를 사용해 성능을 완전히 제어하려면 스칼라와 자바 같은 정적 데이터 타입의 언어를 사용하는 것을 추천합니다.
반면 파이썬이나 R은 각 언어가 제공하는 강력한 라이브러리를 활용하려는 경우에 사용하는 것이 좋습니다.

## Test framework
코드를 단위 테스트하려면 각 언어의 표준 프레임워크(예: JUnit 또는 ScalaTest)를 사용하고 테스트 하네스에서 테스트마다 SparkSession을 생성하고 제거하도록 설정하는 것이 좋습니다.

## Data source
비즈니스 로직을 가진 함수가 데이터소스에 직접 접근하지 않고 DataFrame이나 Dataset을 넘겨받도록 합니다.
스파크의 구조적 API를 사용하는 경우 이름이 지정된 테이블(named table)을 이용해 문제를 해결할 수 있습니다.



## Performance tuning

## DataFrame vs SQL vs Dataset vs RDD
모든 언어에서 DataFrame, Dataset 그리고 SQL의 속도는 동일합니다.
즉, DataFrame은 어떤 언어에서 사용하더라도 성능은 동일합니다.
하지만 파이썬이나 R을 사용해 UDF를 정의하면 성능 저하가 발생할 수 있으므로 자바와 스칼라를 사용해 UDF를 정의하는 것이 좋습니다. 근본적으로 성능을 개선하고 싶다면 UDF 대신 DataFrame이나 SQL을 사용해야 합니다.
모든 DataFrame, SQL 그리고 Dataset 코드는 RDD로 컴파일됩니다.
하지만 사용자가 직접 RDD 코드를 작성하는 것보다 스파크의 최적화 엔진이 더 ‘나은’ RDD 코드를 만들어냅니다.
RDD를 사용하려면 스칼라나 자바를 사용하기 바랍니다.
만약 스칼라나 자바를 사용할 수 없다면 애플리케이션에서 RDD의 ‘사용 영역’을 최소한으로 제한해야 합니다.
파이썬에서 RDD 코드를 실행한다면 파이썬 프로세스를 오가는 많은 데이터를 직렬화해야 합니다.

## Scheduling
자원을 효율적으로 공유하기 위해 spark.scheduler.mode 속성의 값을 FAIR로 설정하거나 필요한 익스큐터의 코어 수를 --max-executor-cores 인수로 지정하는 것 외에는 방법이 없습니다.
--max-executor-cores 인수로 애플리케이션이 클러스터의 자원을 모두 사용하지 못하도록 막을 수 있습니다.

## File Format
가장 효율적으로 사용할 수 있는 파일 포맷으로 아파치 파케이가 있습니다.
파케이는 데이터를 바이너리 파일에 컬럼 지향 방식으로 저장합니다.
그리고 쿼리에서 사용하지 않는 데이터를 빠르게 건너뛸 수 있도록 몇 가지 '통계'를 함께 저장합니다.

## Partionable and Compressable File
어떤 파일 포맷을 선택하더라도 '분할 가능한' 포맷인지 확인해야 합니다.
분할 가능한 포맷을 사용하면 여러 태스크가 파일의 서로 다른 부분을 동시에 읽을 수 있습니다.
압축 포맷은 분할 가능 여부를 결정하는 주요 요소 중 하나입니다.
ZIP 파일이나 TAR 압축 파일은 분할할 수 없습니다. 그러므로 병렬로 읽을 수 없습니다.
반면 하둡이나 스파크 같은 병렬 처리 프레임워크는 gzip, bzip2 또는 lz4를 이용해 압축된 파일이라면 분할할 수 있습니다.
따라서 자원을 효율적으로 사용하기 위해 입력 데이터를 분할 가능한 포맷으로 만드는 것이 가장 좋습니다.

## Table partitioning
테이블 파티셔닝은 데이터의 날짜 필드 같은 키를 기준으로 개별 디렉터리에 파일을 저장하는 것을 의미합니다.
키를 기준으로 데이터가 분할되었고 특정 범위의 데이터만 필요하다면 스파크는 관련 없는 데이터 파일을 건너뛸 수 있습니다.

## Bucketing
데이터를 버켓팅하면 스파크는 사용자가 조인이나 집계를 수행하는 방식에 따라 데이터를 '사전 분할(pre-partition)'할 수 있습니다.
버켓팅을 사용하면 데이터를 한두 개 파티션에 치우치지 않고 전체 파티션에 균등하게 분산시킬 수 있습니다.

예를 들어 데이터를 읽은 다음 특정 컬럼으로 자주 조인한다면 버켓팅을 사용해 조인 컬럼의 값에 따라 데이터를 분할할 수 있습니다.
이렇게 하면 조인 전에 발생하는 셔플을 미리 방지할 수 있으므로 데이터 접근 속도를 높일 수 있습니다.
버켓팅은 물리적 데이터 분할 방법의 보완재로서 보통 파티셔닝과 함께 적용합니다.

## Number of files
데이터를 파티션이나 버켓으로 구성하려면 파일 수와 저장하려는 파일 크기도 고려해야 합니다.
작은 파일이 많은 경우 파일 목록 조회와 파일 읽기 과정에서 부하가 발생합니다.
예를 들어 하둡 분산 파일 시스템(HDFS)에서 데이터를 읽는다면 기본적으로 데이터를 최대 128MB 크기의 블록으로 관리합니다.
5MB 크기의 파일이 30개 있다면 총 150MB가 됩니다.
2개의 HDFS 블록에 데이터를 저장한다고 생각할 수 있지만 총 30개의 블록에 데이터를 저장합니다.

데이터를 효율적으로 저장하려면 입력 데이터 파일이 최소 수십 메가바이트의 데이터를 갖도록 파일의 크기를 조정하는 것이 좋습니다.
maxRecordsPerFile 옵션을 적용하면 write 메서드를 실행할 때 파일당 저장 레코드 수를 지정할 수 있습니다.

## Locality
데이터 지역성은 기본적으로 네트워크를 통해 데이터 블록을 교환하지 않고 특정 데이터를 가진 노드에서 동작할 수 있도록 지정하는 것을 의미합니다.
저장소 시스템이 스파크와 동일한 노드에 있고 해당 시스템이 데이터 지역성 정보를 제공한다면 스파크는 입력 데이터 블록과 최대한 가까운 노드에 태스크를 할당하려 합니다.
스파크 웹 UI에서 데이터 읽기 태스크가 ‘local’로 표시되는 것을 확인할 수 있습니다.

## Statistics
스파크의 구조적 API를 사용하면 비용 기반 쿼리 옵티마이저가 내부적으로 동작합니다.
쿼리 옵티마이저는 입력 데이터의 속성을 기반으로 쿼리 실행 계획을 만듭니다.
하지만 비용 기반 옵티마이저가 작동하려면 정보가 필요합니다.
이를 얻기 위해 사용 가능한 테이블과 관련된 통계를 수집(그리고 유지)해야 합니다.
통계에는 테이블 수준과 컬럼 수준 두 가지 종류가 있습니다.
통계는 이름이 지정된 테이블에서만 사용할 수 있으며, 임의의 DataFrame이나 RDD에서는 사용할 수 없습니다.

컬럼 수준의 통계는 수집하는 데 오래 걸리지만 비용 기반 옵티마이저에서 사용할 수 있는 데이터 컬럼과 관련된 정보를 더 많이 제공합니다. 두 가지 종류의 통계는 조인, 집계, 필터링 그리고 기타 여러 가지 잠재적인 상황(예: 브로드캐스트 조인을 할 때 자동으로 선택)에서 도움이 됩니다.

## Garbage collection
가비지 컬렉션 튜닝의 첫 번째 단계는 가비지 컬렉션의 발생 빈도와 소요 시간에 대한 통계를 모으는 겁니다.
spark.executor.extraJavaOptions 속성에 스파크 JVM 옵션으로 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps 값을 추가해 통계를 모을 수있습니다.
속성값을 설정한 다음 스파크 잡을 실행하면 가비지 컬렉션이 발생할 때마다 워커의로그에 메시지가 출력됩니다.
로그는 드라이버가 아닌 워커 노드의 stdout 파일에 저장됩니다.

태스크가 완료되기 전에 풀 가비지 컬렉션이 자주 발생하면 캐싱에 사용되는 메모리양을 줄여야 합니다(spark.memory.fraction ).
태스크를 처리하기 위한 메모리가 부족하기 때문입니다. -XX:+UseG1GC 옵션을 설정해 G1GC 가비지 컬렉터를 사용할 수 있습니다.
G1GC 가비지 컬렉터는 가비지 컬렉션이 병목 현상을 일으키고 각 영역의 크기를 늘려도 더 이상 부하를 줄일 수 없는 상황에서 성능을 높일 수 있습니다.
익스큐터의 힙 메모리 크기를 늘리려면 -XX:G1HeapRegionSize 옵션을 설정해 G1 영역의 크기를 증가시켜야 합니다.

## Parallelism
spark.default.parallelism과 spark.sql.shuffle.partitions의 값을 클러스터 코어 수에 따라 설정합니다.
스테이지에서 처리해야 할 데이터양이 매우 많다면 클러스터의 CPU 코어당 최소 2~3개의 태스크를 할당합니다.

## Advanced filtering
스파크 잡에서 데이터 필터링 과정을 가장 먼저 수행하는 방법은 자주 사용하는 성능 향상 기법입니다.
상황에 따라 데이터소스에 필터링을 위임하여 최종 결과와 무관한 데이터를 스파크에서 읽지 않고 작업을 진행할 수 있습니다.
파티셔닝과 버켓팅 기법을 활용하는 것 또한 성능 향상에 많은 도움이 됩니다.
최대한 이른 시점에 많은 양의 데이터를 필터링할 수 있다면 스파크 잡은 더 빠르게 수행됩니다.

## Repartition and Coalesce
파티션 재분배 과정은 셔플을 수반합니다.
하지만 클러스터 전체에 걸쳐 데이터가 균등하게 분배되므로 잡의 전체 실행 단계를 최적화할 수 있습니다.
일반적으로 가능한 한 적은 양의 데이터를 셔플하는 것이 좋습니다.
그러므로 셔플 대신 동일 노드의 파티션을 하나로 합치는 coalesce 메서드를 실행해 DataFrame이나 RDD의 전체 파티션 수를 먼저 줄여야 합니다.
이보다 느린 repartition 메서드는 부하를 분산하기 위해 네트워크로 데이터를 셔플링합니다.
파티션 재분배는 조인이나 cache 메서드 호출 시 매우 유용합니다.

## UDF
DF 사용을 최대한 피하는 것도 아주 좋은 최적화 방법입니다.
UDF는 데이터를 JVM 객체로 변환하고 쿼리에서 레코드당 여러 번 수행되므로 많은 자원을 소모합니다.
따라서 사용자가 직접 UDF를 작성하는 것보다 구조적 API를 최대한 활용해 데이터를 처리하는 것이 좋습니다.

## Caching
캐싱은 클러스터의 익스큐터 전반에 걸쳐 만들어진 임시 저장소(메모리나 디스크)에 DataFrame, 테이블 또는 RDD를 보관해 빠르게 접근할 수 있도록 합니다.
캐싱이 필요한 상황은 특정 데이터셋(예: DataFrame 또는 RDD)을 다시 사용하려 할 경우입니다.
캐싱은 지연 연산입니다. 따라서 데이터에 접근해야 캐싱이 일어납니다.
RDD API와 구조적 API는 캐싱 수행 방식이 다릅니다.
RDD는 물리적 데이터(즉, bit 값)를 캐시에 저장합니다.
반면 구조적 API의 캐싱은 물리적 실행 계획을 기반으로 이뤄집니다. 객체 참조가 아닌 물리적 실행 계획을 키로 저장하고 처리 과정 동안 물리적 실행 계획을 참조합니다.
원시 데이터를 읽으려 시도했지만 누군가가 먼저 캐시해 놓은 버전의 데이터를 읽으면서 혼란이 발생할 수 있습니다.

## Join
동등 조인은 최적화하기 가장 쉬우므로 우선적으로 사용하는 것이 좋습니다.
그 외에도 조인 순서를 변경하는 간단한 작업만으로도 성능을 크게 높일 수 있습니다.
조인 순서 변경은 내부 조인을 사용해 필터링하는 것과 동일한 효과를 누릴 수 있습니다.
브로드캐스트 조인 힌트를 사용하면 스파크가 쿼리 실행 계획을 생성할 때 지능적으로 계획을 세울 수 있습니다.
한편 안정성과 최적화를 위해 카테시안 조인이나 전체 외부 조인의 사용을 최대한 피해야 합니다.
마지막으로 테이블 통계와 버켓팅은 조인에 상당한 영향을 미칠 수 있습니다.
조인 전에 테이블 통계를 수집하면 스파크가 조인 타입을 결정하는 데 유용하게 사용됩니다.
그리고 데이터를 적절하게 버켓팅하면 조인 수행 시 거대한 양의 셔플이 발생하지 않도록 미리 방지할 수 있습니다.

## Aggregation
집계 전에 충분히 많은 수의 파티션을 가질 수 있도록 데이터를 필터링하는 것이 최선의 방법입니다.
RDD를 사용하면 집계 수행 방식을 정확하게 제어하고 코드의 성능과 안정성을 개선할 수 있습니다(예: 가능하면 groupByKey 대신 reduceByKey를 사용합니다).

## Broadcast
사용자 애플리케이션에서 사용되는 다수의 UDF에서 큰 데이터 조각을 사용한다면 이 데이터 조각을 개별 노드에 전송해 읽기 전용 복사본으로 저장합니다.
이 기능을 활용하면 잡마다 데이터 조각을 재전송하는 과정을 건너뛸 수 있습니다.
예를 들어 브로드캐스트 변수는 룩업 테이블이나 머신러닝 모델을 저장하는 데 사용할 수 있습니다.