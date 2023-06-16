<img src="https://companieslogo.com/img/orig/MDB_BIG-ad812c6c.png?t=1648915248" width="50%" title="Github_Logo"/>

# MongoDB Atlas Handson Workshop for Samsung Heavy Industry

## Index and Aggreagation
Time Series Collection을 정의 하고 데이터를 저장 하여 봅니다.

### [&rarr; Define Timeseries Collection](#Timeseries)

### [&rarr; Insert Data into Timeseries](#Insert)

### [&rarr; Data Find](#Find)


<br>

### Timeseries
Time Series는 시계열성 데이터를 다루기 위한 것으로 IoT성격의 장비/설비로 부터 발생되는 데이터를 다루기 위한 것으로 자주 사용 됩니다. 시계열 데이터를 다루기 때문에 데이터가 생성되는 Time stamp가 반드시 필요하며 장비/설비에 대한 ID와 해당 시점에서 측정 (measure)되는 데이터로 구성이 됩니다.   
데이터 사용은 주로 시간을 기준으로 하여 데이터를 조회 하게됩니다. 시계열을 기준으로 데이터가 정렬 되어 있음으로 시간 범위를 지정하는 경우  정렬된 데이터를 읽기 때문에 속도가 매우 빠른 편입니다.   

컬렉션을 생성 하기 위해새 IoT로 부터 생성되는 데이터를 먼저 정의해 봅니다. 
다음과 같이 time stamp 데이터를 저장하고 있는 필드 (timestamp)와 데이터를 생산하는 주체로 설비 (equip)과 센서 (sensor)를 정의 합니다. 측정되는 데이터 값은 온도, 습도, 압력 등이 측정 된다고 가정 합니다. 생성되는 데이터 샘플은 다음과 같습니다.  

````
{
    timestamp: ISODate("2023-06-15T06:00:00.000Z"),
    metadata:{
        equip_id: "SHI001",
        sensor_id: "SHISENSOR1",
    },
    temperature: 30,
    humidity: 40,
    pressure: 100
}
````
timestamp, metadata를 제외한 필드는 측정 필드 (measure)가 됩니다.   

Time Series를 위한 컬렉션을 정의 합니다. 일반 MongoDB 컬렉션은 별도의 정의 없이 사용이 가능하지만 Time Series의 경우 특수한 형태의 컬렉션으로 사전에 컬렉션을 정의 해주는 것이 필요 합니다.   

````
db.createCollection(
    <<COLLECTION NAME>>,
    {
       timeseries: {
          timeField: <<TIME STAMP FIELD>>,
          metaField: <<META DATA FIELD>>,
          granularity: <<granularity>>
       }
    }
)
````
사전에 정의한 데이터로 컬렉션을 정의하면 다음과 같습니다.

````
mdb1 [primary] shi> db.createCollection("shipTime", {timeseries:{timeField:"timestamp",metaField:"metadata",granularity:"seconds"}} )
````
컬렉션 이름은 shipTime 이며 시간 데이터가 들어가는 필드(timestamp), 메타데이터가 들어가는 필드의 명 (metadata)를 입력 하여 줍니다.    
granularity 는 시계열의 발생 주기로 초단위로 발생하는 데이터의 경우 "seconds"를 입력 하고 분단위로 발생 하는 경우 "minutes" 을 지정하고 시간단위로 발생 하면 "hours"를 입력 하여 줍니다.   
timeField, metaField, granularity의 데이터는 String 타입만을 가져 갈 수 있습니다. (배열로 여러개 값을 지정 할 수 없습니다.)   

### Insert
데이터를 생성 하여 입력 하도록 합니다. 예제로 생성한 것과 유사하게 데이터를 생성합니다. 다음과 같이 mongosh에서 MQL을 수행 하여 줍니다.

````
mdb1 [primary] shi> let i=0;

mdb1 [primary] shi> let temperature = 0;

mdb1 [primary] shi> let humidity=0;

mdb1 [primary] shi> let pressure=0;

mdb1 [primary] shi> for (i = 0; i < 60; i++) 
{
    temperature = Math.floor(Math.random() * 30);
    humidity = Math.floor(Math.random() * 80);
    pressure = Math.floor(Math.random() * 100);
    let timestamp="";
    if (i < 10) {timestamp = ISODate("2023-06-15T06:0" + i + ":00.000Z");} 
    else {timestamp = ISODate("2023-06-15T06:" + i + ":00.000Z");} 
    
    let insert = { timestamp: timestamp, metadata: { equip_id: "SHI001", sensor_id: "SHISENSOR1" }, measure: { temperature: temperature, humidity: humidity, pressure: pressure } }; 
    db.shipTime.insertOne(insert); 
    db.ship.insertOne(insert);
}
````

두개 컬렉션 shipTime과 비교를 위한 일반 컬렉션 ship에 동일한 데이터가 생성 됩니다.

60개의 데이터가 생성이 됩니다.

다음은 24시간으로 하여 생성한 것으로 전체 데이터가 44,640건(한달 31일 * 하루 24시간 * 60분)이며 9개 Sensor 로 401,760 건의 데이터가 생성된 것입니다.   

<img src="/05.timeseries/images/image01.png" width="90%" height="90%">   

데이터 용량은 56.71MB인 것을 확인 할 수 있습니다.

다음은 timeseries 컬렉션에 동일한 것을 생성 한 것으로 용량이 1.8MB인것을 확인 할 수 있습니다.

<img src="/05.timeseries/images/image02.png" width="90%" height="90%">   

데이터 건수도 다음과 같이 동일 하게 401,760건이 생성된 것을 확인 할 수 있습니다.    

````
mdb1 [primary] shi> db.shipTime.find({}).count()
4320
mdb1 [primary] shi> 
````

### Find

검색을 위한 인덱스를 생성 합니다. 기존 ship 의 경우 다음과 같이 인덱스를 생성 하여 줍니다. 

````
mdb1 [primary] shi> db.ship.createIndex({"metadata.equip_id":1,"metadata.sensor_id":1,"timestamp":1})
metadata.equip_id_1_metadata.sensor_id_1_timestamp_1
````

동일한 인덱스를 time series 컬렉션에도 생성 하여 줍니다.
````
mdb1 [primary] shi> db.shipTime.createIndex({"metadata.equip_id":1,"metadata.sensor_id":1,"timestamp":1})
metadata.equip_id_1_metadata.sensor_id_1_timestamp_1
````

일반 MQL을 그대로 사용 할 수 있습니다.

````
mdb1 [primary] shi> db.shipTime.find({"metadata.equip_id":"SHI001","metadata.sensor_id":"SHISENSOR3"}).explain("executionStats")
{
  explainVersion: '1',
  stages: [
    {
      '$cursor': {
        queryPlanner: {
          namespace: 'shi.system.buckets.shipTime',
          indexFilterSet: false,
          parsedQuery: {
            '$and': [
              { 'meta.equip_id': { '$eq': 'SHI001' } },
              { 'meta.sensor_id': { '$eq': 'SHISENSOR3' } }
            ]
          },
          queryHash: '952037B7',
          planCacheKey: '1DEEA462',
          maxIndexedOrSolutionsReached: false,
          maxIndexedAndSolutionsReached: false,
          maxScansToExplodeReached: false,
          winningPlan: {
            stage: 'FETCH',
            filter: { 'meta.sensor_id': { '$eq': 'SHISENSOR3' } },
            inputStage: {
              stage: 'IXSCAN',
              keyPattern: {
                'meta.equip_id': 1,
                'control.min.netadata.sensor_id': 1,
                'control.max.netadata.sensor_id': 1,
                'control.max.timestamp': -1,
                'control.min.timestamp': -1
              },
              indexName: 'metadata.equip_id_1_netadata.sensor_id_1_timestamp_-1',
              isMultiKey: false,
              multiKeyPaths: {
                'meta.equip_id': [],
                'control.min.netadata.sensor_id': [],
                'control.max.netadata.sensor_id': [],
                'control.max.timestamp': [],
                'control.min.timestamp': []
              },
              isUnique: false,
              isSparse: false,
              isPartial: false,
              indexVersion: 2,
              direction: 'forward',
              indexBounds: {
                'meta.equip_id': [ '["SHI001", "SHI001"]' ],
                'control.min.netadata.sensor_id': [ '[MinKey, MaxKey]' ],
                'control.max.netadata.sensor_id': [ '[MinKey, MaxKey]' ],
                'control.max.timestamp': [ '[MaxKey, MinKey]' ],
                'control.min.timestamp': [ '[MaxKey, MinKey]' ]
              }
            }
          },
          rejectedPlans: []
        },
        executionStats: {
          executionSuccess: true,
          nReturned: 45,
          executionTimeMillis: 37,
          totalKeysExamined: 405,
          totalDocsExamined: 405,
          executionStages: {
            stage: 'FETCH',
            filter: { 'meta.sensor_id': { '$eq': 'SHISENSOR3' } },
            nReturned: 45,
            executionTimeMillisEstimate: 0,
            works: 406,
            advanced: 45,
            needTime: 360,
            needYield: 0,
            saveState: 1,
            restoreState: 1,
            isEOF: 1,
            docsExamined: 405,
            alreadyHasObj: 0,
            inputStage: {
              stage: 'IXSCAN',
              nReturned: 405,
              executionTimeMillisEstimate: 0,
              works: 406,
              advanced: 405,
              needTime: 0,
              needYield: 0,
              saveState: 1,
              restoreState: 1,
              isEOF: 1,
              keyPattern: {
                'meta.equip_id': 1,
                'control.min.netadata.sensor_id': 1,
                'control.max.netadata.sensor_id': 1,
                'control.max.timestamp': -1,
                'control.min.timestamp': -1
              },
              indexName: 'metadata.equip_id_1_netadata.sensor_id_1_timestamp_-1',
              isMultiKey: false,
              multiKeyPaths: {
                'meta.equip_id': [],
                'control.min.netadata.sensor_id': [],
                'control.max.netadata.sensor_id': [],
                'control.max.timestamp': [],
                'control.min.timestamp': []
              },
              isUnique: false,
              isSparse: false,
              isPartial: false,
              indexVersion: 2,
              direction: 'forward',
              indexBounds: {
                'meta.equip_id': [ '["SHI001", "SHI001"]' ],
                'control.min.netadata.sensor_id': [ '[MinKey, MaxKey]' ],
                'control.max.netadata.sensor_id': [ '[MinKey, MaxKey]' ],
                'control.max.timestamp': [ '[MaxKey, MinKey]' ],
                'control.min.timestamp': [ '[MaxKey, MinKey]' ]
              },
              keysExamined: 405,
              seeks: 1,
              dupsTested: 0,
              dupsDropped: 0
            }
          }
        }
      },
      nReturned: Long("45"),
      executionTimeMillisEstimate: Long("0")
    },
    {
      '$_internalUnpackBucket': {
        exclude: [],
        timeField: 'timestamp',
        metaField: 'metadata',
        bucketMaxSpanSeconds: 86400,
        assumeNoMixedSchemaData: true
      },
      nReturned: Long("44640"),
      executionTimeMillisEstimate: Long("29")
    }
  ],
  serverInfo: {
    host: 'mdb.subnet11111545.vcn11111545.oraclevcn.com',
    port: 27001,
    version: '6.0.6',
    gitVersion: '26b4851a412cc8b9b4a18cdb6cd0f9f642e06aa7'
  },
  serverParameters: {
    internalQueryFacetBufferSizeBytes: 104857600,
    internalQueryFacetMaxOutputDocSizeBytes: 104857600,
    internalLookupStageIntermediateDocumentMaxSizeBytes: 104857600,
    internalDocumentSourceGroupMaxMemoryBytes: 104857600,
    internalQueryMaxBlockingSortMemoryUsageBytes: 104857600,
    internalQueryProhibitBlockingMergeOnMongoS: 0,
    internalQueryMaxAddToSetBytes: 104857600,
    internalDocumentSourceSetWindowFieldsMaxMemoryBytes: 104857600
  },
  command: {
    aggregate: 'system.buckets.shipTime',
    pipeline: [
      {
        '$_internalUnpackBucket': {
          timeField: 'timestamp',
          metaField: 'metadata',
          bucketMaxSpanSeconds: 86400,
          assumeNoMixedSchemaData: true,
          usesExtendedRange: false
        }
      },
      {
        '$match': {
          'metadata.equip_id': 'SHI001',
          'metadata.sensor_id': 'SHISENSOR3'
        }
      }
    ],
    cursor: {},
    collation: {}
  },
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1686892029, i: 1 }),
    signature: {
      hash: Binary(Buffer.from("983f66abe8d5e85a832de695ad5cd73dd477bcfd", "hex"), 0),
      keyId: Long("7243648398392295429")
    }
  },
  operationTime: Timestamp({ t: 1686892029, i: 1 })
}
````
Timeseries의 실행 결과로 센서와 설비 정보에 대해 모든 데이터를 조회 하는 것으로 37ms가 소요된 것을 알 수 있습니다.   
일반 컬렉션에서 동일한 쿼리를 수행 하는 경우  결과는 다음과 같습니다.   

````
mdb1 [primary] shi> db.ship.find({"metadata.equip_id":"SHI001","metadata.sensor_id":"SHISENSOR3"}).explain("executionStats")
{
  explainVersion: '1',
  queryPlanner: {
    namespace: 'shi.ship',
    indexFilterSet: false,
    parsedQuery: {
      '$and': [
        { 'metadata.equip_id': { '$eq': 'SHI001' } },
        { 'metadata.sensor_id': { '$eq': 'SHISENSOR3' } }
      ]
    },
    queryHash: '5B64697A',
    planCacheKey: '5C436B9F',
    maxIndexedOrSolutionsReached: false,
    maxIndexedAndSolutionsReached: false,
    maxScansToExplodeReached: false,
    winningPlan: {
      stage: 'FETCH',
      inputStage: {
        stage: 'IXSCAN',
        keyPattern: {
          'metadata.equip_id': 1,
          'metadata.sensor_id': 1,
          timestamp: 1
        },
        indexName: 'metadata.equip_id_1_metadata.sensor_id_1_timestamp_1',
        isMultiKey: false,
        multiKeyPaths: {
          'metadata.equip_id': [],
          'metadata.sensor_id': [],
          timestamp: []
        },
        isUnique: false,
        isSparse: false,
        isPartial: false,
        indexVersion: 2,
        direction: 'forward',
        indexBounds: {
          'metadata.equip_id': [ '["SHI001", "SHI001"]' ],
          'metadata.sensor_id': [ '["SHISENSOR3", "SHISENSOR3"]' ],
          timestamp: [ '[MinKey, MaxKey]' ]
        }
      }
    },
    rejectedPlans: []
  },
  executionStats: {
    executionSuccess: true,
    nReturned: 44640,
    executionTimeMillis: 57,
    totalKeysExamined: 44640,
    totalDocsExamined: 44640,
    executionStages: {
      stage: 'FETCH',
      nReturned: 44640,
      executionTimeMillisEstimate: 8,
      works: 44641,
      advanced: 44640,
      needTime: 0,
      needYield: 0,
      saveState: 44,
      restoreState: 44,
      isEOF: 1,
      docsExamined: 44640,
      alreadyHasObj: 0,
      inputStage: {
        stage: 'IXSCAN',
        nReturned: 44640,
        executionTimeMillisEstimate: 5,
        works: 44641,
        advanced: 44640,
        needTime: 0,
        needYield: 0,
        saveState: 44,
        restoreState: 44,
        isEOF: 1,
        keyPattern: {
          'metadata.equip_id': 1,
          'metadata.sensor_id': 1,
          timestamp: 1
        },
        indexName: 'metadata.equip_id_1_metadata.sensor_id_1_timestamp_1',
        isMultiKey: false,
        multiKeyPaths: {
          'metadata.equip_id': [],
          'metadata.sensor_id': [],
          timestamp: []
        },
        isUnique: false,
        isSparse: false,
        isPartial: false,
        indexVersion: 2,
        direction: 'forward',
        indexBounds: {
          'metadata.equip_id': [ '["SHI001", "SHI001"]' ],
          'metadata.sensor_id': [ '["SHISENSOR3", "SHISENSOR3"]' ],
          timestamp: [ '[MinKey, MaxKey]' ]
        },
        keysExamined: 44640,
        seeks: 1,
        dupsTested: 0,
        dupsDropped: 0
      }
    }
  },
  command: {
    find: 'ship',
    filter: {
      'metadata.equip_id': 'SHI001',
      'metadata.sensor_id': 'SHISENSOR3'
    },
    '$db': 'shi'
  },
  serverInfo: {
    host: 'mdb.subnet11111545.vcn11111545.oraclevcn.com',
    port: 27001,
    version: '6.0.6',
    gitVersion: '26b4851a412cc8b9b4a18cdb6cd0f9f642e06aa7'
  },
  serverParameters: {
    internalQueryFacetBufferSizeBytes: 104857600,
    internalQueryFacetMaxOutputDocSizeBytes: 104857600,
    internalLookupStageIntermediateDocumentMaxSizeBytes: 104857600,
    internalDocumentSourceGroupMaxMemoryBytes: 104857600,
    internalQueryMaxBlockingSortMemoryUsageBytes: 104857600,
    internalQueryProhibitBlockingMergeOnMongoS: 0,
    internalQueryMaxAddToSetBytes: 104857600,
    internalDocumentSourceSetWindowFieldsMaxMemoryBytes: 104857600
  },
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1686891979, i: 1 }),
    signature: {
      hash: Binary(Buffer.from("e312f0b74cb25a15f9234fdf12f24fd9b7c45ed5", "hex"), 0),
      keyId: Long("7243648398392295429")
    }
  },
  operationTime: Timestamp({ t: 1686891979, i: 1 })
}
````

수행 시간이 57ms가 소요 된 것을 확인 할 수 있습니다. 