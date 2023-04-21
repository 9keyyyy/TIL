# JPA Auditing - BaseTimeEntity

## 📍 JPA Auditing 사용하기
- jpa가 제공하는 audit 기능으로, 시간에 자동으로 값을 넣어줌 > 자동으로 시간을 맵핑해 데이터베이스의 테이블에 넣어줌
- 데이터의 생성시간, 수정시간, 혹은 생성한 사람 등을 저장해야할 때
- 매번 모든 엔티티에 컬럼으로 지정해서 코드를 작성하기에 번거로움


### BaseTimeEntity
```java
package com.sejongmate.common.domain;

import jakarta.persistence.Column;
import jakarta.persistence.EntityListeners;
import jakarta.persistence.MappedSuperclass;
import lombok.Getter;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;

@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public class BaseTimeEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;

    @LastModifiedDate
    private LocalDateTime lastModifiedDate;
}

```
- `@MappedSuperclass` : Jpa 엔티티 클래스가 BaseTimeEntity를 상속하는 경우 해당 클래스가 가진 필드도 열로 인식되도록 함
- `@EntityListeners(AuditingEntityListener.class)` : Auditing 기능 제공
- `@CreatedDate` : 생성 시간 자동 저장
- `@LastModifiedDate` : 변경 시간 자동 저장
- 원하는 Entity 클래스에 위의 클래스 상속한 후, main application 클래스에 반드시 `@EnableJpaAuditing` 어노테이션을 추가해주어야 함

