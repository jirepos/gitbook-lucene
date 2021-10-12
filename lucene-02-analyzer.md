# 분석기(Analyzer)

Lucene Analyzer는 문서를 인덱싱하고 검색하는 동안 텍스트를 분석하는 데 사용됩니다. 이 자습서 에서는 일반적으로 사용되는 분석기, 사용자 지정 분석기를 구성하는 방법 및 다양한 문서 필드에 대해 다른 분석기를 할당하는 방법에 대해 설명합니다.

## 루신 분석기

Lucene Analyzer는 텍스트를 토큰으로 분할합니다. **분석기는 주로 토크나이저와 필터로 구성됩니다**. 다양한 분석기는 토크나이저와 필터의 다양한 조합으로 구성됩니다. 일반적으로 사용되는 분석기 간의 차이점을 보여주기 위해 다음 방법을 사용합니다.

FIELD_NAME은 아무거나 관계없다.

```java
private static final String FIELD_NAME = "1sssampleName";

public List<String> analyze(String text, Analyzer analyzer) throws IOException{
    List<String> result = new ArrayList<String>();
    TokenStream tokenStream = analyzer.tokenStream(FIELD_NAME, text);
    CharTermAttribute attr = tokenStream.addAttribute(CharTermAttribute.class);
    tokenStream.reset();
    while(tokenStream.incrementToken()) {
       result.add(attr.toString());
    }       
    return result;
}
```

## 일반적인 루신 분석기

이제 일반적으로 사용되는 Lucene 분석기를 살펴보겠습니다.

### 표준 분석기

가장 일반적으로 사용되는 분석기인 StandardAnalyzer 부터 시작하겠습니다.

```java
private static final String SAMPLE_TEXT
  = "This is baeldung.com Lucene Analyzers test";

@Test
public void testStandardAnalyze() throws IOException {
  List<String> result = analyze(SAMPLE_TEXT, new StandardAnalyzer());
  for (String item : result) {
    System.out.println(item);
  }
}
```

StandardAnalyzer가 URL과 이메일을 인식 할 수 있습니다. 또한 중지 단어(Stop Word)를 제거하고 생성된 토큰을 소문자로 지정합니다. 출력된 결과는 다음과 같다.

```
this
is
baeldung.com
lucene
analyzers
test
```

### StopAnalyzer

StopAnalyzer는 LetterTokenizer, LowerCaseFilter, and StopFilter로 구성되어 있다.

```java
  @Test
  public void testStopAnalyzer() throws IOException {
    CharArraySet stopWords = StopFilter.makeStopSet("is", "com", "lucene");
    List<String> result = analyze(SAMPLE_TEXT, new StopAnalyzer(stopWords));
    for (String item : result) {
      System.out.println(item);
    } 
  }
```

결과는 다음과 같다.

```
this
baeldung
analyzers
test
```

이 예에서 LetterTokenizer는 문자가 아닌 문자로 텍스트를 분할하고 StopFilter 는 토큰 목록에서 중지 단어를 제거합니다. 그러나, StandardAnalyzer와는 달리 StopAnalyzer는 URL을 인식 할 수 없습니다.

### 공백분석기

WhitespaceAnalyzer는 whitespace characters에 의해 텍스트를 분할하는 WhitespaceTokenizer 만 사용한다.

```java
  @Test
  public void testWhiteSpaceAnalyzer() throws IOException {
    List<String> result = analyze(SAMPLE_TEXT, new WhitespaceAnalyzer());
    for (String item : result) {
      System.out.println(item);
    }
  }
```

결과는 다음과 같다.

```
This
is
baeldung.com
Lucene
Analyzers
test
```

### KeywordAnalyzer

KeywordAnalyzer는 입력(input)을 단일 토큰으로 분할한다.

```java
  @Test
  public void testKeywordAnalyzer() throws IOException {
    List<String> result = analyze(SAMPLE_TEXT, new KeywordAnalyzer());
    for (String item : result) {
      System.out.println(item);
    }
  }
```

결과는 다음과 같다.

```
This is baeldung.com Lucene Analyzers test
```

### Language Analyzers

EnglishAnalyzer , FrenchAnalyzer 및 SpanishAnalyzer 와 같은 다양한 언어를 위한 특수 분석기도 있습니다 .

```java
  @Test
  public void testEnglishAnalyzer() throws IOException {
    List<String> result = analyze(SAMPLE_TEXT, new EnglishAnalyzer());
    for (String item : result) {
      System.out.println(item);
    }
  }
```

결과는 다음과 같다.

```
baeldung.com
lucen
analyz
test
```

## Custom Analyzer

다음으로 맞춤형 분석기를 구축하는 방법을 살펴보겠습니다. 우리는 두 가지 다른 방법으로 동일한 사용자 정의 분석기를 구축할 것입니다.

첫 번째 예에서는 CustomAnalyzer 빌더를 사용하여 사전 정의된 토크나이저 및 필터에서 분석기를 구성합니다

```java
  @Test
  public void testCustomAnalyzerBuilder() throws IOException {
    Analyzer analyzer = CustomAnalyzer.builder()
        .withTokenizer("standard")
        .addTokenFilter("lowercase")
        .addTokenFilter("stop")
        .addTokenFilter("porterstem")
        .addTokenFilter("capitalization")
        .build();
    List<String> result = analyze(SAMPLE_TEXT, analyzer);
    for (String item : result) {
      System.out.println(item);
    }
  }
```

결과는 다음과 같다.

```
Baeldung.com
Lucen
Analyz
Test
```

우리의 분석기는 EnglishAnalyzer 와 매우 유사 하지만 대신 토큰을 대문자로 사용합니다.

두 번째 예제에서는 Analyzer 추상 클래스 를 확장 하고 createComponents() 메서드를 재정 의하여 동일한 분석기를 빌드합니다 .

```java
package com.example;


import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.CharArraySet;
import org.apache.lucene.analysis.LowerCaseFilter;
import org.apache.lucene.analysis.StopFilter;
import org.apache.lucene.analysis.TokenStream;
import org.apache.lucene.analysis.en.PorterStemFilter;
import org.apache.lucene.analysis.miscellaneous.CapitalizationFilter;
import org.apache.lucene.analysis.standard.StandardTokenizer;


public class MyCustomAnalyzer extends Analyzer  {
  @Override
  protected TokenStreamComponents createComponents(String fieldName) {
      CharArraySet stopWords = StopFilter.makeStopSet("is", "com", "lucene");
      StandardTokenizer src = new StandardTokenizer();
      //TokenStream result = new StandardFilter(src); // StandardFilter deprecated
      TokenStream result = new LowerCaseFilter(src);
      result = new StopFilter(result,  stopWords);
      result = new PorterStemFilter(result);
      result = new CapitalizationFilter(result);
      return new TokenStreamComponents(src, result);
  }
}
```

TestCase를 다음과 같이 작성했지만 InMemoryLuceneIndex가 사라져서 테스트를 할 수 없었다.

```java
  @Test
  public void givenTermQuery_whenUseCustomAnalyzer_thenCorrect() {
    // InMemoryLuceneInde 클래스도 사라져서 테스트할 수 없음
        // InMemoryLuceneIndex luceneIndex = new InMemoryLuceneIndex(new RAMDirectory(), new MyCustomAnalyzer());
    // luceneIndex.indexDocument("introduction", "introduction to lucene");
    // luceneIndex.indexDocument("analyzers", "guide to lucene analyzers");
    // Query query = new TermQuery(new Term("body", "Introduct"));

    // List<Document> documents = luceneIndex.searchIndex(query);
    // assertEquals(1, documents.size());
  }
```

### References

[https://www.baeldung.com/lucene-analyzers#common](https://www.baeldung.com/lucene-analyzers#common)
