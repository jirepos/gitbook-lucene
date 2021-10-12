# 시작하기

루씬은 주어진 텍스트를 검색을 위한 작은 조각으로 변환하는 분석이라는 과정(Analysis)을 통해 인덱스를 생성하고 인덱스 파일들을 디렉터리에 저장한다. 분석을 처리하는 것을 Analyzer라고 하는데 인덱스를 생성할 때 사용한 Analyzer를 검색할 때에도 사용한다.

## Maven Setup

pom.xml에 의존성을 추가한다.

```markup
  <dependencies>
    <dependency>
      <groupId>org.apache.lucene</groupId>
      <artifactId>lucene-core</artifactId>
      <version>8.9.0</version>
    </dependency>
    <dependency>
      <groupId>org.apache.lucene</groupId>
      <artifactId>lucene-analyzers-common</artifactId>
      <version>8.9.0</version>
    </dependency>
    <dependency>
      <groupId>org.apache.lucene</groupId>
      <artifactId>lucene-queryparser</artifactId>
      <version>8.9.0</version>
    </dependency>

  </dependencies>
```

## 기본 컨셉

### Document

Document는 fields의 Collection이다. 각 필드는 그것과 관련된 값을 가지고 있다. 인덱스는 하나 이상의 document로 구성된다. 검색 결과는 최적으로 매칭된 문서들의 셋트들이다. 항상 plain text document를 의미하지 않는다. database table이거나 collection 일 수 있다.

### Fields

Document는 Field data를 가질 수 있다. where(그곳에서) field는 전형적으로 data value를 가지는 key이다.

```
title: Goodness of Tea
body: Discussing goodness of drinking herbal tea...
```

title과 body는 함께 또는 필드이고 단독 또는 개별적으로 검색될 수 있다.

### Analysis

Analysis는 주어진 text를 더 작은 정밀한 단위로 변환하는 것이다.

### Searching

인덱스가 생성되면, Query 와 IndexSearcher를 사용하여 index를 검색할 수 있다.

> IndexWriter는 index를 생성할 책임이 있다. IndexSearcher는 인색스를 검색할 책임이 있따.

### Query Syntax

루신은 매우 다이나믹하고 쓰기에 쉬운 query syntax를 제공한다.

특정 필드에 text를 검하려면

```
fieldName:text
eg: title:tea
```

Ranges Search

```
timestamp:[1509909322,1572981321]
```

wildcards를 사용할 수 있다.

```
dri?nk
d*k
```

### Simple Application

간단한 애플리케이션을 만들 것이다. 몇 개의 문서들을 인덱싱할 것이다.

먼저, in-memory index를 생성하고 거기에 몇 개의 문서들을 추가할 것이다. 인덱스를 생성하기 위해서 IndexWriter를 사용하는데, IndexWriter는 설정을 위해 IndexWriterConfig를 필요로 한다. 분석작업을 할 Analyzer가 필요한데, 여기서는 StandardAnalyzer를 사용한다.

```java
Directory index = new RAMDirectory();
StandardAnalyzer analyzer = new StandardAnalyzer();
IndexWriterConfig config = new IndexWriterConfig(analyzer);
IndexWriter w = new IndexWriter(index, config);
```

이제 TextField를 가진 document를 생성한다. IndexWriter를 사용하여 index에 그것들을 추가한다. 마지막에 IndexWriter를 닫는다.

```java
Document doc = new Document();
doc.add(new TextField("title", title, Field.Store.YES));
doc.add(new StringField("isbn", isbn, Field.Store.YES));
w.addDocument(doc);
w.close();
```

이제 검색 쿼리(search query)를 생성하고 추가된 문서(document)에 대한 인덱스를 검색해보자.

검색 쿼리를 생성하기 위해서 QueryParaser를 사용한다. 'title' 인자는 쿼리에서 명시된 필드가 없을 때 사용할 디폴트 필드이다. Index를 읽기 위해서 IndexReader를 사용하는데 인덱스를 생성할 때 사용했던 Directory 인스턴스를 사용한다. IndexSearcher를 사용하여 검색을 실행한다.

```java
String querystr = "lucene";
Query q = new QueryParser("title", analyzer).parse(querystr);

int hitsPerPage = 10;
IndexReader reader = DirectoryReader.open(index);
IndexSearcher searcher = new IndexSearcher(reader);
TopDocs docs = searcher.search(q, hitsPerPage);
ScoreDoc[] hits = docs.scoreDocs;
```

search() 메서드가 반화난 TopDocs를 사용하여 docId를 구하고 IndexSearcher.doc()를 사용하여 Document를 구한다. 마지막을 IndexReader를 close한다.

```java
System.out.println("Found " + hits.length + " hits.");
for(int i=0;i<hits.length;++i) {
   int docId = hits[i].doc;
   Document d = searcher.doc(docId);
   System.out.println((i + 1) + ". " + d.get("isbn") + "\t" + d.get("title"));
}
reader.close();
```

아래는 RamDirectory를 사용한 전체코드이다.

```java
package com.example;

import java.io.IOException;

import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.StringField;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexReader;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.queryparser.classic.ParseException;
import org.apache.lucene.queryparser.classic.QueryParser;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.Query;
import org.apache.lucene.search.ScoreDoc;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.RAMDirectory;

public class HelloLucene {
    public static void main(String[] args) throws IOException, ParseException {
        // 0. Specify the analyzer for tokenizing text.
        //    The same analyzer should be used for indexing and searching
        StandardAnalyzer analyzer = new StandardAnalyzer();

        // 1. create the index
        Directory index = new RAMDirectory();

        IndexWriterConfig config = new IndexWriterConfig(analyzer);

        IndexWriter w = new IndexWriter(index, config);
        addDoc(w, "Lucene in Action", "193398817");
        addDoc(w, "Lucene for Dummies", "55320055Z");
        addDoc(w, "Managing Gigabytes", "55063554A");
        addDoc(w, "The Art of Computer Science", "9900333X");
        w.close();

        // 2. query
        String querystr = args.length > 0 ? args[0] : "lucene";

        // the "title" arg specifies the default field to use
        // when no field is explicitly specified in the query.
        Query q = new QueryParser("title", analyzer).parse(querystr);

        // 3. search
        int hitsPerPage = 10;
        IndexReader reader = DirectoryReader.open(index);
        IndexSearcher searcher = new IndexSearcher(reader);
        TopDocs docs = searcher.search(q, hitsPerPage);
        ScoreDoc[] hits = docs.scoreDocs;

        // 4. display results
        System.out.println("Found " + hits.length + " hits.");
        for(int i=0;i<hits.length;++i) {
            int docId = hits[i].doc;
            Document d = searcher.doc(docId);
            System.out.println((i + 1) + ". " + d.get("isbn") + "\t" + d.get("title"));
        }

        // reader can only be closed when there
        // is no need to access the documents any more.
        reader.close();
    }

    private static void addDoc(IndexWriter w, String title, String isbn) throws IOException {
        Document doc = new Document();
        doc.add(new TextField("title", title, Field.Store.YES));

        // use a string field for isbn because we don't want it tokenized
        doc.add(new StringField("isbn", isbn, Field.Store.YES));
        w.addDocument(doc);
    }
}
```

## Referecnes

[https://www.baeldung.com/lucene](https://www.baeldung.com/lucene)\
[ https://devyongsik.tistory.com/349](https://devyongsik.tistory.com/349) [https://tourspace.tistory.com/237](https://tourspace.tistory.com/237) [http://www.lucenetutorial.com/lucene-in-5-minutes.html](http://www.lucenetutorial.com/lucene-in-5-minutes.html)
