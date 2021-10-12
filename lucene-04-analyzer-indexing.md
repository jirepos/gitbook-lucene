# 인덱싱 옵션

보통의 인덱싱은 Document 클래스를 이용하여 다음과 같이 지정한다.

```java
Document document = new Document();
document.add(new Field(“area”, area, Store.YES, Index.UN_TOKENIZED));
document.add(new Field(“vmag”, vmag, Store.YES, Index.UN_TOKENIZED));
document.add(new Field(“count”, count, Store.YES, Index.NO));
document.add(new Field(“data”, data, Store.YES, Index.NO));
writer.addDocument(document);[/code]
```

정확히는 Field 객체를 넣는것이다.

```
new Field(필드명, 저장할 값, 저장옵션, 인덱스옵션)
```

**필드명** 데이터베이스의 컬럼이라고할 수 있다. 어떤 이름의 필드에 저장할 것인지를 지정한다.

**저장할 값** 지정한 해당 필드명에 들어갈 값을 지정한다.

**저장옵션**

* Store.YES: 인덱스를 할 값 모두를 인덱스에 저장한다. 검색결과등에서 꼭 보여야 하는 내용이라면 사용한다.
* Store.NO: 값을 저장하지 않는다. Index 옵션과 혼합하여, 검색은 되지만, 원본글이 필요없을 경우 사용될수 있다.
* Store.COMPRESS: 값을 압축하여 저장한다. 저장할 글의 내용이 크거나, 2진 바이너리 파일등에 사용한다.

**인덱스 옵션**\
검색을 위한 인덱스 생성 방식을 정한다.

* Index.NO : 인덱싱을 하지 않는다. 이렇게 저장한 값으로 검색을 할수 없다.
* Index.TOKENIZED : Analyzer에 의한 토크나이즈를 수행하여 인덱싱을 한다. 검색 가능하다.
* Index.UN_TOKENIZED : 토크나이즈를 수행하지 않는다. 숫자라거나, 쪼갤필요가 없는 문자열에 사용하면 된다. 검색이 되며, Analyze를 수행하지 않기때문에 인덱스 속도가 빠르다.
* Index.NO_NORMS : 이것은 인덱싱 시간이 매우 빨라야 할때 사용한다. Analyze를 수행하지 않는다. 또한 필드 Length 노멀라이즈를 수행하지 않는다. 인덱싱시에 적은 메모리만을 사용하게 된다는 장점이 있다.

**옵션 사용 예**

* 저장할 내용은 수필형식의 글이다. 해당 필드로 검색이 되어야 하며, 검색결과에서 바로 해당 글의 전문이 출력되어야 한다. => Store.YES, Index.TOKENIZED
* 저장할 내용은 20070606 같은 형식을 가지는 띄어쓰기가 없는 날짜이다. 해당 필드로 검색이 되어야 하며, 검색어와 검색결과가 동일할테니 글의 내용을 저장할 필요는 없다. => Store.NO Index.Index.UN_TOKENIZED
* 저장할 내용은 영화 제목과 영화 사용기이다. 영화 제목만으로 검색을 할수 있으며, 사용기의 전문으로 검색은 되지 않는다. 하지만 검색결과에서 사용기가 출력되어야 한다. => 제목 Field : Store.YES, Index.TOKENIZED / 사용기 Field : Store.YES, Index.NO
