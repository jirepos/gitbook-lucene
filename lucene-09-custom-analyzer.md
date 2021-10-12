# 사용자 분석기

## 사용자 분석기 생성

커스텀 애널라이저를 생성한다.

```java
package com.example.lucene;

import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.LowerCaseFilter;
import org.apache.lucene.analysis.TokenStream;
import org.apache.lucene.analysis.ngram.NGramTokenFilter;
import org.apache.lucene.analysis.standard.StandardTokenizer;


/**
 * Tokenizer는 StandardTokenizer를 사용하고 NGramTokenFilter를 사용하는 커스텀 Analyzer이다. 
 */
public class NgramAnalyzer extends Analyzer {
  @Override
  protected TokenStreamComponents createComponents(String fieldName) {
    //StopFilter –> LowerCaseFilter –> KoreanFilter  –> KoreanTokenizer
      //NGramTokenizer tokenizer = new NGramTokenizer(2,20);
      StandardTokenizer tokenizer = new StandardTokenizer();
      //KoreanTokenizer tokenizer = new KoreanTokenizer();
      TokenStream tokenStream = new LowerCaseFilter(tokenizer);
      //List<String> stopWords = Arrays.asList( "<", "/>");
      //final CharArraySet stopSet = new CharArraySet(stopWords, true);
      //tokenStream = new StopFilter(tokenStream, StopFilter.makeStopSet(stopWords));
      //tokenStream = new StopFilter(tokenStream, stopSet);
      tokenStream = new NGramTokenFilter(tokenStream, 2, 24, false);
      //tokenStream = new KoreanNumberFilter(tokenStream);
      return new TokenStreamComponents(tokenizer, tokenStream);
  }
}///~
```

## 검색을 위한 공통 클래스 만들기

### 상수값 정의

검색 클래스에서 사용할 상수를 정의한다.

```java
package com.example.lucene;
/**
 * 검색에서 사용할 상수 
 */
public class LuceneConstant {
  /** 색인 디렉터리 */
  public static final String INDEX_PATH = "E:/lucene";
}//~
```

### 인덱스 생성 클래스

인덱스를 생성하는 기능을 제공하기 위해 클래스를 만든다.

```java
package com.example.lucene;

import java.io.File;
import java.io.IOException;
import java.nio.file.Paths;

import org.apache.lucene.document.Document;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.index.IndexWriterConfig.OpenMode;
import org.apache.lucene.index.Term;
import org.apache.lucene.store.FSDirectory;

/**
 * 인덱스를 생성하는 기능을 제공하고 삭제 및 갱신을 제공한다. 
 */
public class LuceneIndexer {

  private NgramAnalyzer analyzer; 
  private IndexWriter indexWriter;
  private FSDirectory dir;
  private OpenMode openMode;

  /**
   * 생성자 
   * @param openMode  IndexWriter OpenMode 
   * @throws Exception
   */
  public LuceneIndexer(OpenMode openMode) throws Exception {
    this.openMode = openMode; 
    this.analyzer = new NgramAnalyzer();
    createIndexWriter();
  }


  /**
   * IndexWriter 인스턴스 생성
   * @throws Exception
   */
  private void createIndexWriter() throws Exception {
    dir = FSDirectory.open(Paths.get(LuceneConstant.INDEX_PATH));
    IndexWriterConfig config = new IndexWriterConfig(analyzer);
    // OpenMode.CREATE - 색인시마다 기존 색인 삭제 후 재 색인
    // OpenMode.CREATE_OR_APPEND - 기존 색인이 없으면 만들고, 있으면 append 함. 
    // OpenMode.APPEND - 기존 색인에 추가
    config.setOpenMode(this.openMode);
    //config.setOpenMode(OpenMode.CREATE);
    //TieredMergePolicy mergePolicy = new TieredMergePolicy();
    //config.setMergePolicy(mergePolicy);
    //checkLock();
    this.indexWriter = new IndexWriter(dir, config);
  }//:


  /**
   * 인덱스를 갱신한다. 
   * @param document  갱신할 문서 
   * @throws Exception
   */
  public void update(Document document) throws Exception {
    String postId = document.get("id");
    Term term = new Term("id", postId); 
    this.indexWriter.updateDocument(term, document); 
  }//:

  /**
   * 인덱스에서 문서 삭제 
   * @param postId 삭제할 아이디 
   * @throws Exception
   */
  public void delete(String id) throws Exception  { 
    this.indexWriter.deleteDocuments(new Term("id", id));
    // 삭제 대상이 있는지 확인 
    this.indexWriter.hasDeletions();
    this.indexWriter.commit();
  }



  /**
   * 인덱싱할 문서 추가 
   * @param d Document 인스턴스 
   * @throws IOException
   */
  public void add(Document d) throws IOException {
    this.indexWriter.addDocument(d);
  }

  /**
   * IndexWriter를 닫는다. 
   * @throws IOException
   */
  public void close() throws IOException{ 
    this.indexWriter.close();
  }



  /**
   * write.lock 파일이 있는지 체크. 8.9 버전에서는 의미가 없는 것 같으나 
   * 정확히 파악되지 않음. 
   * @throws IOException
   * @throws InterruptedException
   */
  private void checkLock() throws IOException, InterruptedException {
    //IndexWriter.WRITE_LOCK_NAME - 실제 index 시 directory에 보면 xxx.lock 파일이 존재하게 되는데 
    //존재 할 경우는 lock으로 판단. 
    File file = new File(LuceneConstant.INDEX_PATH + "/" + IndexWriter.WRITE_LOCK_NAME );
    //while(dir.fileExists(IndexWriter.WRITE_LOCK_NAME)){
    while(file.exists()){
      System.out.println("write.lock file exsists.");
    // dir.clearLock(name);
        Thread.sleep(10);
    }
  }
}///~
```

### 검색을 위한 클래스 생성

인덱스를 검색할 검색클래스를 생성한다.

```java
package com.example.lucene;

import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

import org.apache.lucene.document.Document;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexReader;
import org.apache.lucene.index.Term;
import org.apache.lucene.search.BooleanClause;
import org.apache.lucene.search.BooleanQuery;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.Query;
import org.apache.lucene.search.ScoreDoc;
import org.apache.lucene.search.TermQuery;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.search.TopScoreDocCollector;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.FSDirectory;


/**
 * 인덱스를 검색하는 기능을 제공한다. 
 */
public class LuceneSearcher {

  /**
   * 인덱스 검색 
   * @param options 검색 조건 
   * @param page  페이지 번호 
   * @param n 페이지당 가져올 갯 수 
   * @return
   *   검색된 문서 목록 
   * @throws Exception
   */
  public static SearchResult search(SearchOption options, int page, int n ) throws Exception {
    // StandardAnalyzer analyzer = new StandardAnalyzer();
    //NgramAnalyzer analyzer = new NgramAnalyzer();
    Directory indexDirectory = FSDirectory.open(Paths.get(LuceneConstant.INDEX_PATH));
    IndexReader indexReader = DirectoryReader.open(indexDirectory);
    IndexSearcher searcher = new IndexSearcher(indexReader);
    //Query query = new QueryParser(inField, analyzer).parse(queryString);

    BooleanQuery.Builder builder = new BooleanQuery.Builder();
    Optional<List<String>> should = Optional.ofNullable(options.getShould());
    if(should.isPresent()) {
      // OR 검색 조건 
      for(String m : should.get() ) {
        BooleanQuery.Builder builder2 = new BooleanQuery.Builder();
        Optional<List<String>> searchFields = Optional.of(options.getSearchFields());
        for(String field : searchFields.get()) {
          Query q = new TermQuery(new Term(field, m));  
          builder2.add(q, BooleanClause.Occur.SHOULD);
        }
        builder.add(builder2.build(), BooleanClause.Occur.SHOULD);
      }
    }

    Optional<List<String>> must = Optional.ofNullable(options.getMust());
    if(must.isPresent()) {
      // AND 검색 조건 
      for(String m : must.get() ) {
        BooleanQuery.Builder builder2 = new BooleanQuery.Builder();
        Optional<List<String>> searchFields = Optional.of(options.getSearchFields());
        for(String field : searchFields.get()) {
          Query q = new TermQuery(new Term(field, m));  
          builder2.add(q, BooleanClause.Occur.SHOULD);
        }
        builder.add(builder2.build(), BooleanClause.Occur.MUST);
      }
    }

    if(n <= 0) { n = 10;}
    //TopDocs topDocs = searcher.search(builder.build(), n);
    TopDocs topDocs = searchPaging(builder.build(), searcher, page, n); 
    //searcher.searchAfter(after, query, n, sort);
    System.out.println("총 갯수:" + topDocs.totalHits); 
    // List<ScoreDoc> list = Arrays.asList(topDocs.scoreDocs);
    List<Document> list = new ArrayList<Document>();
    for (ScoreDoc sdoc : topDocs.scoreDocs) {
      Document d = searcher.doc(sdoc.doc);
      list.add(d);
    }

    indexReader.close();
    SearchResult result = new SearchResult();
    result.setTotalHits(topDocs.totalHits.value);
    result.setDocuments(list);
    return result; 
    // return topDocs.scoreDocs.stream()
    // .map(scoreDoc -> searcher.doc(scoreDoc.doc))
    // .collect(Collectors.toList());
  }//:



  /**
   * Paging 처리 
   * @param query Query 인스턴스 
   * @param searcher IndexSearcher 인스턴스
   * @param page  페이지 번호 
   * @param n 가져올 갯 수 
   * @return
   * @throws Exception
   */
  private static TopDocs searchPaging(Query query, IndexSearcher searcher, int page, int n) throws Exception  {
    // https://stackoverflow.com/questions/963781/how-to-achieve-pagination-in-lucene/963828
    if(n <= 0) { n = 10;}
    int startIndex = (page -1) * n; 
    // Sort sort = new Sort(
    //   new SortField("updDate", SortField.Type.STRING, true /* reverse */)
    // );
    TopScoreDocCollector collector = TopScoreDocCollector.create(9999, 1); // 최대 9999개 
    searcher.search(query, collector);
    TopDocs topDocs = collector.topDocs(startIndex, n);  // startIndex부터 n 개 
    return topDocs; 
  }//:


}///~
```

### 검색 옵션을 제공할 클래스

검색 옵션을 제공할 클래스르 생성한다.

```java
package com.example.lucene;

import java.util.List;

import lombok.Getter;
import lombok.Setter;

/**
 * 쿼리 옵션을 제공한다. 
 */
@Getter
@Setter
public class SearchOption {
  /** 반드시 포함해야 하는 검색어를 담는다. */
  private List<String> must; 
  /** OR 조건으로 사용할 검색어를 담는다. */
  private List<String> should; 
  /** 폼함하지 않아야할 검색어를 담는다.  */
  private List<String> mustNot;
  /** 검색 필드를 설정한다.  */
  private List<String> searchFields; 
}
```

### 검색 결과를 제공할 클래스 생성

검색결과를 담을 클래스를 생성한다.

```java
package com.example.lucene;

import java.util.List;

import org.apache.lucene.document.Document;

import lombok.Getter;
import lombok.Setter;

/**
 * 검색결과 
 */
@Setter
@Getter
public class SearchResult {
  /** 전체 검색 결과 건수  */
  private long totalHits; 
  /** 검색된 결과  */
  private List<Document> documents;
}
```

## 색인을 위한 래핑 클래스 생성

블로그 포스트를 색인할 클래스를 생성한다.

```java
package com.example.blog;

import java.util.Date;

import com.example.lucene.LuceneIndexer;

import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.IndexWriterConfig.OpenMode;

/**
 * Blog 인덱스 생성을 한다. 
 */
public class BlogIndexer {
  /** Indexer  */
  private LuceneIndexer indexer = null;
  /**
   * 생성자 
   * @param openMode OpenMode 값 
   * @throws Exception
   */
  public BlogIndexer(OpenMode openMode) throws Exception  {
    this.indexer = new LuceneIndexer(openMode); 
  }
  /**
   * IndexWriter를 닫는다. 
   * @throws Exception
   */
  public void close() throws Exception  {
    this.indexer.close();
  }
  /**
   * 문서를 추가한다  
   * @param id id 필드 값 
   * @param title title 필드 값 
   * @param contents  contents 필드 값 
   * @param author author 필드 값 
   * @param updDate  updDate 필드 값 
   * @throws Exception
   */
  public void addDocument(String id, String title, String contents, String author, Date updDate) throws Exception   {
    Document document = new Document();
    document.add(new TextField("id", id, Field.Store.YES));
    document.add(new TextField("title", title, Field.Store.YES));
    document.add(new TextField("author", author, Field.Store.YES));
    document.add(new TextField("contents", contents, Field.Store.YES));
    document.add(new TextField("updDate", updDate.toInstant().toString(), Field.Store.YES));

    indexer.add(document);
  }
}/// ~
```

## 색인하기

다음과 같이 사용한다.

```java
  /** 인덱스 생성 테스트 케이스 */
  @Test
  public void testLuceneIndexer() throws Exception  {
    int seq = 1000;
    String id = null; 
    String title = null; 
    String contents = null; 
    BlogIndexer indexer = new BlogIndexer(OpenMode.CREATE);
    for(int i=0; i < 900; i++) {
      id = String.valueOf(seq); 
      title = "주간보고 올려주세요.";
      contents = "잘 살아보자. 주간보고. 일일보고";
      indexer.addDocument(id, title, contents, "홍길동", Date.from(Instant.now()));
      seq++; 
    }
    indexer.close();
    System.out.println("Indexing closed.");
  }
```

## 검색하기

다음과 같이 검색을 할 수 있다.,

```java
  /** 검색 테스트 케이스  */
  @Test
  public void testSearch() throws Exception  {
    //String queryStr = "홍길동";
    //BlogSearch.search(queryStr);
    //List<Document> list = BlogSearch.searchFiles("title", queryStr);
    // List<Document> list = BlogSearch.searchFiles2(queryStr);
    // for(Document d : list) {
    //   System.out.println(d.get("title"));
    // }
    List<String> searchFields = Arrays.asList("title", "author", "contents");
    List<String> must = Arrays.asList("주간보고");
    //List<String> should = Arrays.asList("홍길동", "주간보고");
    List<String> should = Arrays.asList("홍길동", "박신혜");
    SearchOption options = new SearchOption();
    options.setMust(must);
    options.setShould(should);
    options.setSearchFields(searchFields);
    SearchResult result  = LuceneSearcher.search(options, 2, 10);
    List<Document> list = result.getDocuments();
    for(Document d : list) {
      System.out.println(d.get("id") + "/" + d.get("title") + "/" + d.get("contents") + "/" + d.get("author") + "/" + d.get("updDate"));
    }
  }//;
```

## 색인 삭제

다음과 같이 색인을 삭제할 수 있다.

```java
  /**
   * 인덱스 삭제 테스트 케이스 
   * @throws Exception
   */
  @Test
  public void testDeleteDoc() throws Exception { 
    String id = "1111";
    LuceneIndexer indexBase = new LuceneIndexer(OpenMode.CREATE_OR_APPEND); 
    indexBase.delete(id);
  }//:
```

## 색인 갱신

다음과 같이 색인을 갱신할 수 있다.

```java
  /** 인덱스 업데이트 테스트 케이스  */
  @Test
  public void testUpdateDoc() throws Exception { 
    String id = "1111";
    LuceneIndexer indexBase = new LuceneIndexer(OpenMode.CREATE_OR_APPEND); 
    indexBase.delete(id);
  }//:
```
