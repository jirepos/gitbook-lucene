# FSDirectory

저장 방식에는 많이 쓰는 방식이 몇 가지를 지원하는데 NIODirectory는 NUnix계열에서만 가능 하지만 버그가 있는 것으로 알려졌다. FSDirectory를 가장 많이 쓴다고 한다. 파일을 인덱싱하려면 먼저 파일 시스템 인덱스를 만들어야 합니다. Lucene은 파일 시스템 인덱스를 생성하기 위해 FSDirectory 클래스를 제공합니다 :

```java
Directory directory = FSDirectory.open(Paths.get(indexPath));
```

여기서 indexPath 는 디렉토리의 위치입니다. 디렉토리가 존재하지 않으면 Lucene이 생성합니다. Lucene은 추상 FSDirectory 클래스 의 세 가지 구체적인 구현인 SimpleFSDirectory, NIOFSDirectory 및 MMapDirectory를 제공합니다. 그들 각각은 주어진 환경에 특별한 문제가 있을 수 있습니다.

예를 들어 SimpleFSDirectory 는 여러 스레드가 동일한 파일에서 읽을 때 차단하므로 동시 성능이 좋지 않습니다.

마찬가지로 NIOFSDirectory 및 MMapDirectory 구현은 Windows의 파일 채널 문제와 메모리 릴리스 문제에 각각 직면합니다.

이러한 환경적 특성을 극복하기 위해 Lucene은 FSDirectory.open() 메서드를 제공합니다 . 호출되면 환경에 따라 최상의 구현을 선택하려고 합니다.

* OpenMode.CREATE - 색인시마다 기존 색인 삭제 후 재 색인
* OpenMode.CREATE_OR_APPEND - 기존 색인이 없으면 만들고, 있으면 append 함. 
* OpenMode.APPEND - 기존 색인에 추가.

lucene은 rdb와 다르게 lock에 대한 처리를 안해준다. 단지, index시 lock 파일이 존재하면 error가 발생한다. 이 때문에 필히 lock 체크를 해줘야함.

> IndexWriter를 close해도 lock file이 삭제되지 않는다. lock file을 체크하지 않아도 정상동작을 하는데 lock 파일을 lucene이 어떻게 처리하는지 아직 정확히 파악되지 않았다.

```java
//IndexWriter.WRITE_LOCK_NAME - 실제 index 시 directory에 보면 xxx.lock 파일이 존재하게 되는데 
        //존재 할 경우는 lock으로 판단. 
        while(dir.fileExists(IndexWriter.WRITE_LOCK_NAME)){
//            dir.clearLock(name);
            Thread.sleep(10);
        }

`
```

## References

[https://zest133.tistory.com/entry/lucene-%EA%B8%B0%EC%B4%88-index](https://zest133.tistory.com/entry/lucene-%EA%B8%B0%EC%B4%88-index)\
[https://www.baeldung.com/lucene-file-search](https://www.baeldung.com/lucene-file-search)
