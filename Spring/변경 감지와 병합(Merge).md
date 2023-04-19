# 변경 감지와 병합

## 📍 준영속 엔티티
- 영속성 컨텍스트가 더이상 관리하지 않는 엔티티
- jpa가 관리하고 있지 않음

### 준영속 엔티티를 수정하는 방법
1. 변경 감지 기능 사용
2. 병합(`merge`) 사용

</br>
아래 예제를 통해 위의 방법을 적용해보면,

## 📍 상품 수정 코드
### **Controller**
```java
@PostMapping("items/{itemId}/edit")
public String updateItem(@PathVariable("itemId") Long itemId, @ModelAttribute("form") BookForm form) {

    Book book = new Book();

    book.setId(form.getId());
    book.setName(form.getName());
    book.setPrice(form.getPrice());
    book.setStockQuantity(form.getStockQuantity());
    book.setAuthor(form.getAuthor());
    book.setIsbn(form.getIsbn());

    itemService.saveItem(book);
    return "redirect:/items";
}
```
- 위의 book은 준영속 상태의 엔티티 
- db에 한번 갔다옴. 과거에 jpa가 관리한 적 있으나, 더이상 관리하고 있지는 않음
- jpa가 식별할 수 있는 id를 가지고 있음 (임의로 만들어낸 엔티티도 식별자를 가지고 있으면 준영속 엔티티로 볼 수 있음)
- 위 코드는 병합을 통해 데이터를 변경하는 예시. 아래의 `saveItem()` 코드 참고

</br>

### **Service**
```java
@Transactional
public void saveItem(Item item){
    itemRepository.save(item);
}

//변경 감지에 의해 데이터를 변경하는 방법
@Transactional
public Item updateItem(Long itemId, String name, int price, int stockQuantity){
    Item findItem = itemRepository.findOne(itemId);
    findItem.setPrice(price);
    findItem.setName(name);
    findItem.setStockQuantity(stockQuantity);
    // itemRepository.save(findItem); 코드 필요 x. Transactional에 의해 코드가 커밋되고 디비에 반영

    return findItem;
}
```
- 위의 `updateItem` 함수는 **변경 감지**에 의해 데이터를 변경하는 방법
- 영속성 컨텍스트에서 엔티티를 다시 조회한 후 데이터를 수정하는 방법임
- 위의 `finditem`은 영속 엔티티
- 트랜잭션 안에서 엔티티를 다시 조회 + 변경할 값 선택 > 커밋 시점에 변경감지(dirty checking)해서 데이터베이스에 업데이트 
- 위 코드처럼 set을 계속 사용하는 것보다 `change` 메서드를 선언하여 엔티티에서 수정할 수 있도록 하는 것이 좋음 (비지니스 로직을 엔티티 레벨에)
- 즉 컨트롤러는 아래와 같이 사용하는 것이 좋음
    ```java
        @PostMapping("items/{itemId}/edit")
    public String updateItem(@PathVariable("itemId") Long itemId, @ModelAttribute("form") BookForm form) {

        itemService.updateItem(itemId, form.getName(), form.getPrice(), form.getStockQuantity());

        return "redirect:/items";
    }
    ```



</br>

### **Repository**
```java
public void save(Item item){
    if(item.getId() == null){   // 새로 생성하는 객체(신규 등록)
        em.persist(item);
    }else{  // 이미 등록된 객체(업데이트)
        em.merge(item);
    }
}
```
- **병합**을 사용해 데이터를 변경하는 방법
- 병합은 준영속 상태의 엔티티를 영속 상태로 변경할 때 사용하는 기능
- `merge`는 영속성 컨텍스트에서 엔티티를 찾고 파라미터로 넘긴 값을 해당 엔티티에 바꿔치기 하는 것. 이후 트랜젝션 커밋할 때 반영됨
- 즉, 위의 `updateItem` 함수의 코드를 jpa가 코드 한줄로 해주는 것

</br>

## 📍 병합
<img width="814" alt="스크린샷 2023-04-19 오후 5 09 35" src="https://user-images.githubusercontent.com/62213813/233011125-dcc16c08-4673-46f1-afc8-541dd5a60bcd.png">

1. `merge()` 호출
2. 파라미터로 넘어온 준영속 엔티티의 식별자 값으로 1차 캐시에서 엔티티 조회 (캐시에 없으면 데이터베이스에서 엔티티 조회 > 1차 캐시에 저장)
3. 조회한 영속 엔티티에 파라미터로 넘어온 엔티티 값을 채워 넣음 
4. 이후 반환 : 반환된 객체는 영속 엔티티이지만, 넘긴 파라미터가 영속성 엔티티가 되는 것은 아님.

</br>

**<주의>** </br>
- 변경 감지 기능을 사용하면 원하는 속성만 선택해서 변경할 수 있지만, 병합을 사용하면 모든 속성이 변경됨
- 즉 병합시 값이 없으면 `null`로 업데이트 할 위험도 있음
### 📌 따라서 병합보다는 변경 감지의 방법으로 데이터를 변경하는 것이 좋음

</br>

종합하면,
- 컨트롤러에서 어설프게 엔티티 만들지 말기
- 트랜잭션이 있는 서비스 계층에 식별자(`id`)와 변경할 데이터를 명확하게 전달하기(파라미터 or dto)
- 트랜잭션이 있는 서비스 계층에서 영속 상태의 엔티티를 조회하고, 엔티티의 데이터를 직접 변경하기 > 트랜잭션 커밋 시점에 변경 감지가 실행됨



</br></br></br></br>

** 실전!스프링 부트와 JPA활용1을 듣고 작성한 내용입니다.