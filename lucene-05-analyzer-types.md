# 분석기 종류

한글이나 일어등의 언어는 공백으로 분리해서 만들어진 term으로는 검색효율이 떨어진다. "사과와 포도는 참 달다"를 \[사과와] \[포도는] \[참] \[달다]와 같이 분리햇을 때 '사과'로 검색하면 결과가 안 나온다. N-Gram 분석기는 글자를 한 글자씩 연결해 가면서 TERM을 생성한다.

## N-Gram Analyzer

N-Gram은 문장을 한글자씩 잘라가면서 n개만큼씩 붙여서 term을 만듭니다.

```
"사과와 포도는 참 달다"

-> N=2
"사과" "과와" "와" "포" "포도" "도는" "참" "달" "달다"

-> N=3
"사과와" "과와" "와 포" "포도" "포도는" "도는 " "는 참" "참" "참 달" "달다"
```

문장을 한글자씩 쪼개서 N개 단위로 붙여서 term을 만든다. N=2일때 "사과"는 검색이 되겠지만 N=3일때는 안될수 있다. 검색되는 확률은 높일 수 있지만 누락되는 검색도 생기고 쓸모 없는 term들도 생긴다. 따라서 검색어를 자르는 용도보다는 특정 패턴을 검색하는데 사용된다.

아래와 Analyzer를 만들어 사용하면 된다.

```java
package com.example;

import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.LowerCaseFilter;
import org.apache.lucene.analysis.TokenStream;
import org.apache.lucene.analysis.ngram.NGramTokenFilter;
import org.apache.lucene.analysis.standard.StandardTokenizer;

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
}
```

NGramTokenizer 생성시 넣는 param값은 min N, max N 입니다. 만약 NGramTokenizer(2,3)으로 만들면 N=2, N=3 일때의 term을 모두 만든다.

> NgramTokenizer를 사용하면 너무 많이 토큰이 발생해서 StandardTokenizer를 사용하고 NGramTokenFilter를 사용하는 것이 좋을 것 같다.

다음과 같이 테스트케이스를 작성한다.

```java
  @Test 
  public void testNgramAnzlyer() throws Exception  {
    // StopFilter –> LowerCaseFilter –> KoreanFilter  –> KoreanTokenizer
    String source = "언어별로 사랑한다는말을하고싶어서 단어의 변화는 모두 다르기 ANAF <html><body></body></html> Love is Blue. 때문에 언어별로 Analyzer를 특화시켜서 사용해야 한다";
    Analyzer analyzer = new NgramAnalyzer();
    TokenStream tokenStream = analyzer.tokenStream("sample", source);
    CharTermAttribute attr1 = tokenStream.addAttribute(CharTermAttribute.class);
    //PositionLengthAttribute attr3 = tokenStream.addAttribute(PositionLengthAttribute.class);
    tokenStream.reset();
    while (tokenStream.incrementToken()) {
      System.out.println(attr1.toString());
    }
    analyzer.close();
  }
```

출력결과는 다음과 같다.

```
언어
언어별
언어별로
어별
어별로
별로
사랑
사랑한
사랑한다
사랑한다는
사랑한다는말
사랑한다는말을
사랑한다는말을하
사랑한다는말을하고
사랑한다는말을하고싶
사랑한다는말을하고싶어
사랑한다는말을하고싶어서
랑한
랑한다
랑한다는
랑한다는말
랑한다는말을
랑한다는말을하
랑한다는말을하고
랑한다는말을하고싶
랑한다는말을하고싶어
랑한다는말을하고싶어서
한다
한다는
한다는말
한다는말을
한다는말을하
한다는말을하고
한다는말을하고싶
한다는말을하고싶어
한다는말을하고싶어서
다는
다는말
다는말을
다는말을하
다는말을하고
다는말을하고싶
다는말을하고싶어
다는말을하고싶어서
는말
는말을
는말을하
는말을하고
는말을하고싶
는말을하고싶어
는말을하고싶어서
말을
말을하
말을하고
말을하고싶
말을하고싶어
말을하고싶어서
을하
을하고
을하고싶
을하고싶어
을하고싶어서
하고
하고싶
하고싶어
하고싶어서
고싶
고싶어
고싶어서
싶어
싶어서
어서
단어
단어의
어의
변화
변화는
화는
모두
다르
다르기
르기
an
ana
anaf
na
naf
af
ht
htm
html
tm
tml
ml
bo
bod
body
od
ody
dy
bo
bod
body
od
ody
dy
ht
htm
html
tm
tml
ml
lo
lov
love
ov
ove
ve
is
bl
blu
blue
lu
lue
ue
때문
때문에
문에
언어
언어별
언어별로
어별
어별로
별로
an
ana
anal
analy
analyz
analyze
analyzer
analyzer를
na
nal
naly
nalyz
nalyze
nalyzer
nalyzer를
al
aly
alyz
alyze
alyzer
alyzer를
ly
lyz
lyze
lyzer
lyzer를
yz
yze
yzer
yzer를
ze
zer
zer를
er
er를
r를
특화
특화시
특화시켜
특화시켜서
화시
화시켜
화시켜서
시켜
시켜서
켜서
사용
사용해
사용해야
용해
용해야
해야
한다
```

## 한글 분석기

언어별로 단어의 변화는 모두 다르기 때문에 언어별로 Analyzer를 특화시켜서 사용해야 한다. 루씬에서는 여러 언어용 analyzer를 제공하기 때문에 해당 언어에 따라 Analyzer를 선택하여 사용하면 된다.

한글은 은전한닢, 꼬꼬마분석기등이 존재하며, lucene 7.4.0 부터 nori 분석기가 포함되었다. 한글 분석기는 문장을 형태소로 쪼개어 term을 생성한다.

pom.xml 파일에 의존성을 추가한다. version은 lucene 버전과 동일한 버전으로 설정한다.

```markup
<dependency>
    <groupId>org.apache.lucene</groupId>
    <artifactId>lucene-analyzers-nori</artifactId>
    <version>8.9.0</version>
</dependency>
```

사용은 new KoreanAnalyzer()로 생성하여 사용하면 된다.

```java
  @Test 
  public void testNori() throws Exception  {
    String source = "언어별로 단어의 변화는 모두 다르기 때문에 언어별로 Analyzer를 특화시켜서 사용해야 한다";
    Analyzer analyzer = new KoreanAnalyzer();
    TokenStream tokenStream = analyzer.tokenStream("sample", source);
    CharTermAttribute attr1 = tokenStream.addAttribute(CharTermAttribute.class);
    //PositionLengthAttribute attr3 = tokenStream.addAttribute(PositionLengthAttribute.class);
    tokenStream.reset();
    while (tokenStream.incrementToken()) {
      System.out.println(attr1.toString());
    }
    analyzer.close();
  }
```

결과는 다음과 같다.

```
언어
단어
변화
다르
때문
언어
analyzer
특화
사용
하
```

## References

[https://tourspace.tistory.com/250?category=888736](https://tourspace.tistory.com/250?category=888736)
