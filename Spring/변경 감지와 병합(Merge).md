# 변경 감지와 병합

## 📍 상품 수정 코드

**Controller**
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

**Service**
```java
@Transactional
public void saveItem(Item item){
    itemRepository.save(item);
}
```

**Repository**
```java
public void save(Item item){
    if(item.getId() == null){   // 새로 생성하는 객체(신규 등록)
        em.persist(item);
    }else{  // 이미 등록된 객체(업데이트)
        em.merge(item);
    }
}
```
