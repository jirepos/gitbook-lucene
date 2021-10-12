# 인덱스 삭제

## 인덱스 삭제

Term을 사용하여 삭제할 Documet를 특정한다음 IndexWriter의 deleteDocuments()를 사용한다. hasDeletions()를 사용하여 삭제대상이 있는지 확인할 수 있다. commit()를 하기 전까지는 완전히 삭제된 것은 아니다. commit()를 사용하여 완전히 삭제한다.

```java
  public void delete(String postId) throws Exception  { 
    this.indexWriter.deleteDocuments(new Term("id", postId));
    // 삭제 대상이 있는지 확인 
    this.indexWriter.hasDeletions();
    this.indexWriter.commit();
  }
```

## 인덱스 업데이트

Term을 이용하여 업데이트할 문서를 특정하고 IndexWriter의 updateDocument를 사용한다.

```java
  public void update(Document document) throws Exception {
    String postId = document.get("id");
    Term term = new Term("id", postId); 
    this.indexWriter.updateDocument(term, document); 
  }//:
```
