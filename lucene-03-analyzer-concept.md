# 분석기 개념

루씬에서 Indexing 작업은 주어진 Text를 Term으로 쪼개서 검색 가능한 최소한의 단위로 만드는 작업이라고 볼수 있다. 그리고, 그 단어들을 검색하기 편하도록 거꾸로 정리 놓는다(Inverted index). 원본 문서를 읽어 의미 단위별로 텍스트를 뽑아주는 작업을 파싱, 또는 전처리라고 부른다. 이런 텍스트를 각자의 필드에 넣고 색인하면 분석 과정을 거쳐 토큰으로 추출한다. 루씬은 텍스트를 텀으로 분리해야 할 필요가 있으면 분석기(Analyzer)를 사용한다. 분석기는 색인하거나 검색할 때 사용한다. 텍스트에서 추출된 토큰은 분석 과정의 기본 단위이다. 색인할 때 지정된 분석기로 토큰을 추출하며, 중요한 몇 가지 속성도 색인에 함께 보관한다. 토큰은 기본적으로 단어와 함께 몇 가지 메타 정보도 담고 있다.

```
This is baeldung Lucene Analyzers test.
```

Analyzer는 위의 텍스르틀 다음과 같은 토큰으로 분리한다.

```
[This] [is] [baldung] [Lucene] [Analyzers] [test]
```

색인 과정에서 이렇게 분리한 토큰에 필드 이름을 더해 텀으로 생성된다. 이렇게 색인된 텀을 검색할 수 있다. 만약 Field.Index.NOT_ANALYZED or Field.Index.ANALYZED_NO_NORMS 설정을 지정하면 분석기를 통하지 않고 전체 텍스트가 하나의 토큰으로 색인된다.

루씬에서는 기본적으로 제공하는 내장 분석기들이 있는데 어떻게 철이하는지 살펴보자.

**whitespaceAnalyzer** 공백 문자 기준으로 토큰을 분리한다.

**SimpleAnalyzer** 알파벳이 아닌 모든 글자 기준으로 토큰을 분리하고, 모두 소문자로 변경한다. 숫자도 모두 제거합니다.

**StopAnalyzer** SimpleAnalyzer와 비슷하지만 추가적으로 불용어를 제거한다. 기본 설정으로 영어의 불용어(the, a등)을 제거하지만, 불용어 목록을 별도로 지정할 수 있다.

**StandardAnalyzer** 루씬에 내장된 분석기 중 가장 섬세한 분석기이다. 텍스트를 분석해 **회사 이름, 이메일 주소, 도메인 이름 등의 다양한 토큰을 인식**하고 별도로 분리한다. 토큰의 알파벳을 모두 소문자로 변경하고, 불용어도 제거하고, 특수 기호도 모두 제거한다.

## Analyzer 개념

크게 나누어 분석은 아래의 단계를 거친다.

```
CharFilter -> Tokenizer -> TokenFilter
```

### CharFilter

HTML이나, 임의의 패턴등을 입력된 text source에서 제거한다. Tokenizer를 수행하기 전에 수행하는 사전 작업으로 이는 토큰으로 나눠지기 전에 offset을 보장하기 위해서이다.

* HTMLStripCharFilter: HTML 구문 제거
* MappingCharFilter: 백스페이스, 탭, 개행문자 등을 유니코드로 변경.

### TokenStream

Source text를 Token으로 나눈 이후 token이 순서대로 나열된 상태를 Token stream 이라고 한다. 실제로 이 작업은 TokenStream class에서 수행된다.

```
This is baeldung Lucene Analyzers test.  => [This] [is] [baldung] [Lucene] [Analyzers] [test]
```

### AttributeSource

AttributeSoucre는 TokenStream의 부모 클래스로 Token의 메타 정보를 담고 있다. 따라서 TokenStream이 생성될때 ArrtibuteSrouce에 보고자 하는 정보를 등록해 놓으면 그때 그때의 snap shot을 확인할 수 있다.

```java
  @Test 
  public void testAttribute() throws Exception  {
    String source = "This is baeldung Lucene Analyzers test.";
    Analyzer analyzer = new StandardAnalyzer();
    TokenStream tokenStream = analyzer.tokenStream("sample", source);
    CharTermAttribute attr1 = tokenStream.addAttribute(CharTermAttribute.class);
    OffsetAttribute attr2   = tokenStream.addAttribute(OffsetAttribute.class);
    //PositionLengthAttribute attr3 = tokenStream.addAttribute(PositionLengthAttribute.class);
    tokenStream.reset();
    while (tokenStream.incrementToken()) {
      System.out.println(attr1.toString());
      System.out.println(attr2.startOffset() + ":" + attr2.endOffset());
    }
    analyzer.close();
  }
```

위 예제에서는 incrementToken() 함수로 token을 하나씩 넘기면서 해당 Token의 정보를 출력한다. 아래는 출력된 결과이다.

```
this
0:4
is
5:7
baeldung
8:16
lucene
17:23
analyzers
24:33
test
34:38
```

### Tokenizer

Source text를 Token으로 분리하며, 주요 Tokenizer는 아래와 같다.

* Standard Tokenizer: lucene core에서 제공하는 기본 analyzer로 유니코드 텍스트 분할.  알고리즘에 명시된 단어 분리 규칙을 따른다.
* Whitespace Tokenizer: 공백 문자로 텍스트를 분할
* NGram Tokenizer: N-Gram 모델을 이용해 분할
* LowerCase Tokenizer: 영문 분할용으로 대문자를 소문자로 바꿔서 분할하며, LetterTokenizer + LowerCaseFitler의 조합
* EdgeNGram Tokenizer: 입력토큰의 시작부분에서 N-Gram을 만듬.

### TokenFilter

Tokenizer에 의해서 분리된 토큰을 정제한다.

* StandardFilter: StandardTokenizer로 추출된 토큰을 표준화 한다. 
* LowerCaseFilter: 소문자로 변환한다.
* StopFilter: 불용어를 제거한다. ex) 'a', 'the' , 'is', 'and' ...

### Analyzer

Analyzer는 Tokenizer와 TokenFitler로 구성된다.실제 Analyzer class는 abstract이며, 각각의 Analyzer는 Analyzer 클래스를 상속받아 아래 abstract 함수를 override한다.

Custom Analyzer를 생성하는 방법은 'Lucene (02) Analyzer' 문서에서 설명하였다.

## References

[https://tourspace.tistory.com/248](https://tourspace.tistory.com/248)
