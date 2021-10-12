# Query

"love"와 같은 문구 하나를 찾기 보다는 여려 조건을 조합해 검색을 할 때가 있다. lucene에서는 기본적인 Term Query 이외에도 다른 query드을 지원한다.

## Lucene에서 제공하는 Query

### Term Query

지금까지 계속사용한 쿼리이다. 검색 할 Term을 설정하여 해당 Filed에서 요청한 문자를 찾는다.

### Boolean query

두 개 이상의 쿼리를 조합하여 사용한다. TermsQuery, PhaseQuery 또는 BooleanQuery를 조합할 수 있다. BooleanQuery 는 builder를 사용하여 작성한다.

```java
fun getBooleanQuery(): BooleanQuery { 
  val query1 = TermQuery(Term("content", "lucene")) 
  val query2 = TermQuery(Term("content", "query")) 
  val booleanQuery = BooleanQuery.Builder() 
    .add(query1, BooleanClause.Occur.MUST) 
    .add(query2, BooleanClause.Occur.MUST_NOT) 
  return booleanQuery.build() 
}
```

* MUST  반드시 포함해야 하는 term
* MUST_NOT  반드시 포함하지 않아야 하는 term
* FILTER  MUST 처럼 반드시 포함해야 하지만 score에는 포함시키지 않음.
* SHOULD  OR의 의미임.  MUST가 없는 BooleanQuery에서 match되려면 document가  반드시 하나 이상의 SHOULD query와 match 되어야함.

### Wildcard query

WildcardQuery는 문자열의 부분 검색을 가능하게 한다. SQL로 따지면 like 검색에서 '%'의 역할이다. 단, wildcard를 사용하면 전체 term을 iteration 하여 검색하므로 속도가 느리다. 검색속도를 극도로 느리게 하지 않기 위해서는 첫글자에 '\*'를 사용하지 않도록 해야 한다.

```java
fun getWildcardQuery(): WildcardQuery { 
  return WildcardQuery(Term("content", "app*")) 
}
```

* \*  character sequence를 대체 
* ? single character를 대체 
* / escape character 

### Prefix query

PrefixQuery는 말 그대로 해당 문구로 시작하는 term을 모두 찾는다.

```java
fun getPrefixQuery(): PrefixQuery {
    return PrefixQuery(Term("content", "app"))
}
```

## BooleanQuery 사용

Blog의 포스트를 인덱싱한다고 가정하자. Post는 4개의 필드가 있다.

```java
public class Post { 
  String postId;
  String title
  String contents;
  String author; 
}
```

"java"라는 검색어로 검색한다고 가정해보자. 검색할 때에는 title이나 contents 필드를 특정하여 검색을 하지 않기 때문에 title 필드나 contents 필드에 "java"가 있는지 모두 검색을 해봐야 한다. 이럴 때에는 다음과 같이 로직을 작성해야 할 것이다.

1. title 필드에 java가 있거나
2. contents 필드에 java가 있거나 

이 두 조건 중 하나가 참이면 인덱싱된 Document는 결과에 표시되어야 한다. 이럴 때 BooleanQuery를 사용한다.

```java
Query q1 = new TermQuery(new Term("title", "java"));
Query q2 = new TermQuery(new Term("contents", "java"));
BooleanQuery.Builder builder = new BooleanQuery.Builder();
builder.add(q1, BooleanClause.Occur.SHOULD);
builder.add(q2, BooleanClause.Occur.SHOULD);
```

"java"라는 문자열이 들어 있고 author가 "홍길동"인 문서를 찾는다고 한다면, 그리고 "홍길동"은 받드시 있어야 한다고 가정한다면 다음과 같이 코드를 작성할 수 있다.

```java
Query q1 = new TermQuery(new Term("title", "java"));
Query q2 = new TermQuery(new Term("contents", "java"));
Query q2 = new TermQuery(new Term("author", "홍길동"));
BooleanQuery.Builder builder = new BooleanQuery.Builder();
builder.add(q1, BooleanClause.Occur.SHOULD);
builder.add(q2, BooleanClause.Occur.SHOULD);
builder.add(q3, BooleanClause.Occur.MUST);
```

좀 더 복잡하게 가보자. "java"가 있거나 "class"가 있는 문서를 찾는다고 가정해 보자.

1. title에 java가 있거나  class가 있거나
2. contents에 java가 있거나 class가 있거나 

첫번째 조건이 참이거나 두번재 조건이 참이거나 둘 중에 하나만 참이어도 검색이 되어야 한다. 이런 경우 다음과 같이 BooleanQuery를 사용할 수 있다.

```java
Query q1 = new TermQuery(new Term("title", "java"));
Query q2 = new TermQuery(new Term("contents", "java"));
BooleanQuery.Builder builder1 = new BooleanQuery.Builder();
builder1.add(q1, BooleanClause.Occur.SHOULD);
builder1.add(q2, BooleanClause.Occur.SHOULD);

Query q1 = new TermQuery(new Term("title", "class"));
Query q2 = new TermQuery(new Term("contents", "class"));
BooleanQuery.Builder builder2 = new BooleanQuery.Builder();
builder2.add(q1, BooleanClause.Occur.SHOULD);
builder2.add(q2, BooleanClause.Occur.SHOULD);


BooleanQuery.Builder builder = new BooleanQuery.Builder();
builder.add(builder1.build(), BooleanClause.Occur.SHOULD);
builder.add(builder2.build(), BooleanClause.Occur.SHOULD);
```

만일, class 문자열이 반드시 있어야 한다면 다음과 같이 수정할 수 있다.

```java
BooleanQuery.Builder builder = new BooleanQuery.Builder();
builder.add(builder1.build(), BooleanClause.Occur.SHOULD);
builder.add(builder2.build(), BooleanClause.Occur.MUST);
```

그러면 "java"라는 문자열은 없을 수도 있지만 "class"라는 문자열은 반드시 있는 문서가 검색된다.

위와 같이 AND 또는 OR 검색을 할 수 있다. IndxexSearcher 객체의 search() 메서드에서 build() 메서드를 사용하여 Query 객체를 파라미터로 전달하여 검색한다.

```java
TopDocs topDocs = indexSearcher.search(builder.build(), 10);
```

> BooleanQuery를 사용할 때에는 TermQuery를 사용할 때와는 다르게 Analyzer를 사용하지 않는다.

아래는 Boolean 쿼리를 사용하여 검색된 문서를 리스트로 반환하는 코드이다.

```java
  public static List<Document> searchFiles2(String queryString) throws Exception {
    // BooleanQuery를 사용할 때에는 Analyzer를 사용하지 않는다 
    // StandardAnalyzer analyzer = new StandardAnalyzer();
    // NgramAnalyzer analyzer = new NgramAnalyzer();

    Directory indexDirectory = FSDirectory.open(Paths.get(INDEX_PATH));
    IndexReader indexReader = DirectoryReader.open(indexDirectory);
    IndexSearcher searcher = new IndexSearcher(indexReader);

    Query q1 = new TermQuery(new Term("title", queryString));
    Query q2 = new TermQuery(new Term("contents", queryString));
    Query q3 = new TermQuery(new Term("author", queryString));

    BooleanQuery.Builder builder = new BooleanQuery.Builder();
    builder.add(q1, BooleanClause.Occur.SHOULD);
    builder.add(q2, BooleanClause.Occur.SHOULD);
    builder.add(q3, BooleanClause.Occur.SHOULD);

    TopDocs topDocs = searcher.search(builder.build(), 10); // 검색한다. 
    List<Document> list = new ArrayList<Document>();
    for (ScoreDoc sdoc : topDocs.scoreDocs) {
      Document d = searcher.doc(sdoc.doc);  // 문서를 찾아서 
      list.add(d);   //목록에 담는다 
    }
    indexReader.close();  // IndexReader를 닫는다 
    return list;   // 목록을 반환한다 
  }//:
```

## 페이징 처리

Pagination을 처리하기 위해서 TopScoreDocCollector를 사용한다. MAX_RESULT는 검색에 조건에 맞는 결과들(hits-DB의 경우 검색된 행들)의 전체 수를 제한하기 위한 정수값이다.

```java
TopScoreDocCollector collector = TopScoreDocCollector.create(MAX_RESULT, true);
```

다음의 예를 살펴보자.

```java
collector = TopScoreDocCollector.create(9999, true);
searcher.search(parser.parse("Clone Warrior"), collector);
// get first page
topDocs = collector.topDocs(0, 10);
int resultSize=topDocs.scoreDocs.length; // 10 or less
int totalHits=topDocs.totalHits; // 9999 or less
```

검색구문 'Clone Warrior'를 포함하는 최대 9999개의 문서들을 수집하라고 Lucene에게 알려준다. 이것은 검색 구문을 포함하는 문서들이 9999개 이상이 있다면 collector가 9999개를 만족한다면 멈춘다는 것을 의미한다.

```java
public TopDocs search(String query, int pageNumber) throws IOException, ParseException {
  Query searchQuery = parser.parse(query);
  TopScoreDocCollector collector = TopScoreDocCollector.create(1000, true);

  int startIndex = (pageNumber - 1) * MyApp.SEARCH_RESULT_PAGE_SIZE;
  searcher.search(searchQuery, collector);

  TopDocs topDocs = collector.topDocs(startIndex, MyApp.SEARCH_RESULT_PAGE_SIZE);
  return topDocs;
}
```

### 기타 페이징 처리

아래의 두 개의 코드들은 다른 페이징 기법들인데 참고하기 위해 기록해 놓았다.

```java
  ScoreDoc[] scoreDocs = results.scoreDocs;
  int pageIndex = [User Value];
  int pageSize = [Configured Value];

  int startIndex = (pageIndex - 1) * pageSize;
  int endIndex = pageIndex * pageSize;
  endIndex = results.totalHits < endIndex? results.totalHits:endIndex;

  for (int i = startIndex ; i < endIndex ; i++)
  {
  luceneDocuments.Add(searcher.Doc(scoreDocs[i].doc));
  }
```

정렬이 필요한 경우에는 다음과 같이 사용할 수 있다고 한다.

```java
val scoredDocs = topDocs1.scoreDocs 
val topDocs2 = searcher.searchAfter(scoredDocs, query, 10, Sort(SortField("contents", SortFiled.Type.STRING)
```

## References

[https://tourspace.tistory.com/244?category=888736](https://tourspace.tistory.com/244?category=888736)\
[https://newbedev.com/lucene-4-pagination](https://newbedev.com/lucene-4-pagination)
