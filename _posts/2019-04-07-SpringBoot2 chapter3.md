---
layout: post
title:  Chapter3 Reactive Data Access with Spring Boot
date:   2019-04-07
author: ag-lee
categories: SpringBootStudy
---



# 3. Reactive Data Access with Spring Boot

* MongoDB를 통해 Reactive 데이터를 어떻게 저장하고 사용할 지에 대해서 배운다.



## Getting underway with a reactive data store

* JPA는 Reactive programming을 지원하지 않기 때문에 mongo db를 이용한다.

* spring boot에서는 spring data mongo db를 지원한다.

    * mongo db core + Reactive stream driver까지 포함한 패키지를 지원
    * `compile('org.springframework.boot:spring-boot-starter-data-mongodb-reactive')`
    * Spring-boot-starter-webflux와 spring-boot-starter-
        Data-mongodb-reactive 가 모두 Project Reactor를 가져오기 때문에 두개를 맞춰줘야한다.  (Dependency-managment가 같은 버전을 가지고 오도록 지원한다.)

    

## Solving a problem

### Spring Data

* Spring Data 예제

    * 아래 코드로 위의 JPA 코드를 사용하는 것처럼 Mongo DB에서 할 수 있다.

    * `ReactiveCrudRepository` 를 상속받아 인터페이스를 작성하면 query를 굳이 작성하지 않아도  CRUD 오퍼레이션을 수행할 수 있다.

        (save, findById, findAll, delete, deleteById, count, exists, …)

    * findByFirstName 같이 메소드 이름을 변경해 find 함수를 커스텀할 수 있다.

```java
interface EmployeeRepository
    extends ReactiveCrudRepository<Employee, Long> {
    Flux<Employee> findByFirstName(Mono<String> name);
}
```

* ßSpring Data는 `Repository` 를 상속한 인터페이스가 있으면 모든 메소드를 탐색하고 method signiture를 파싱한다.

    * 예를 들면, findBy를 발견하면 나머지 메소드 명을 보고 도메인 타입(테이블)에 기초해 프로퍼티 이름을 찾는다.
    * arguments도 같은 방식으로 찾는다. 
    * return type을 보고 어떤 데이터 집합을 만들지 결정한다.

    

* 전체 쿼리는 한번 생성되면 캐싱되어 쿼리를 여러번 사용해도 부하가 없다.

* 인터페이스가 `Spring Data Commons` 를 상속받기 때문에 Mongo DB가 아니라 다른 데이터 저장소를 사용해도 코드를 변경할 필요가 없다.
* 아래 코드를 보면 JPA의 entity 정의지만 Mongo DB에도 사용이 가능하다.
    * `@Data` : lombok annotation. Getters, setters, toString, equals, hashcode 함수를 생성해준다.
    * `@Document` : employees collection에 도메인 오브젝트를 저장해주도록 하는 MongoDB 어노테이션.
    * `@Id` : Spring Data Commons annotation으로 Key를 나타내는 어노테이션.



## Wiring up Spring Data repositories with Spring Boot

* `@EnableReactiveMongoRepositories`  Mongo DB 설정을 활성화 시키는 Spring Data 어노테이션.
    * `@ConditionalOnClass` : 시작할 때 리스트에 있는 클래스들을 classpath에 있어야만 실행된다.
    * `@CondtionalOnMssingBean` : 아래 리스트의 Bean들이 존재하지 않는 경우에만 실행된다.
    * `@ConditionalOnProperty` : 이 설정을 적용하려면, spring.data.mongodb.reactive-repositories의 enabled가 true로 설정되어 있어야 합니다. 이 속성이 없는 경우에는 true로 설정합니다.
    * `@Import` : Reative repositories의 모든 빈 생성을 이 클래스에 위임한다.
    * `@AutoConfigureAfter` : MongoReactiveDataAutoConfiguration 설정이 완료된 후에 이 설정을 적용하면 여기서 설정된 내용을 확신할 수 있습니다.


```java
@Configuration
@ConditionalOnClass({ MongoClient.class, ReactiveMongoRepository.class })
@ConditionalOnMissingBean({
    ReactiveMongoRepositoryFactoryBean.class,
    ReactiveMongoRepositoryConfigurationExtension.class })
@ConditionalOnProperty(prefix = "spring.data.mongodb.reactive-repositories", name = "enabled", havingValue = "true", matchIfMissing = true)
@Import(MongoReactiveRepositoriesAutoConfigureRegistrar.class)
@AutoConfigureAfter(MongoReactiveDataAutoConfiguration.class)
public class MongoReactiveRepositoriesAutoConfiguration {

}
```



* MongoReactiveRepositoriesAutoConfigureRegistrar 가 설정 데이터를 끌어오는데, 이 때 그 클래스의 마지막에는 아래코드가 있다.

```java
@EnableReactiveMongoRepositories
private static class EnableReactiveMongoRepositoriesConfiguration {
}
```

    

* 이 클래스는 우리가 reactive Mongo DB repositories를 사용하지 않아도 Spring Boot가 Reactive MongoDB와 Spring Datat Mongo 2.0+가 있을 때 자동으로 해줄 것입니다.



## Creating a reative repository

* `ReactiveCrudRepository` 로 부터 상속되는 메소드들은 인자로 이미지 타입들을 가질 수도 있고, Mono와 Flux가 상속하는 Publisher<Image>로 받을 수 있다.
* 그리고 Mono와 Flux에 기반한 값을 return값으로도 사용할 수 있다.
    * delete의 경우는 Mono<Void> 를 리턴해 처리가 종료되었는지 여부만을 반환한다. 
    * findById 는 리턴값이 하나이거나 없으므로 Mono<Image>를 반환한다.
    * findAll은 Flux<Image>를 반환한다.



## Pulling data through a Mono/Flux and chain of operations

* Flux는 lazy하게 리턴되기 때문에 요청되었을 때는 이미지들의 수만 메로리로 가지고 온 뒤 나머지는 주어진 시간동안에 가지고 오게 됩니다.

* 저장하는 코드에서 하는 일을 보면,

    * Multipart file들의 flat map을 가지고와서 이미지를 저장하고, 파일을 서버로 복사한다.
    * 파일을 이미지를 모노를 이용해 저장하고, 파일을 모노를 통해 서버에 복사한다. 
    * 그리고 그 결과들을 모노로 받아 Mono.when으로 결과를 join한다. 이것은 DB에 저장되고 파일이 서버에 복사될 때까지 각 파일에 대한 작업이 완료되지 않는 것을 의미한다. 
    * 전체 플로우는 then()으로 종료되고, 각 파일들에 대한 프로세스가 종료된 것을 알 수가 있다.

```java
public Mono<Void> createImage(Flux<FilePart> files) {
            return files
            .flatMap(file -> {
            Mono<Image> saveDatabaseImage = imageRepository.save(
                new Image(
                UUID.randomUUID().toString(),
                    file.filename()));
                Mono<Void> copyFile = Mono.just(
                    Paths.get(UPLOAD_ROOT, file.filename())
                    .toFile())
                    .log("createImage-picktarget")
                    .map(destFile -> {
                        try {
                        destFile.createNewFile();
                        return destFile;
                        } catch (IOException e) {
                            throw new RuntimeException(e);} })
                    .log("createImage-newfile")
                    .flatMap(file::transferTo)
                    .log("createImage-copy");
                return Mono.when(saveDatabaseImage, copyFile);
            }).then(); 
}
```

* Reactor가 Mono.when()으로 묶인 job들이 순서를 보장하진 않지만, 작업을 계속 진행하며 모든 방면을 다 처리하는 것을 보장하기 때문에 여러 개의 task들이 완벽하게 수행해야할 때 유용하다.

    * Mongo db가 다른 일을 처리하거나 외부요인으로 느려질 때 파일을 먼저 저장할 수도 있다. 하지만 중요한건 이 construct가 효율을적이고 결과를 나오게 한다는 것이다.



## Querying by example

* 아래 코드처럼 조건이 여러가지 주어질 때 Spring Data의 함수를 사용하게 되면 코드가 지저분해진다.

```java
interface PersonRepository
        extends ReactiveCrudRepository<Person, Long> {
            List<Person> findByFirstNameAndLastNameAndAgeBetween(
            String firstName, String lastName, int from, int to);
}
```

* `Query by Example` : 필요한 조건을 도메인 오브젝트에 모아서 쿼리를 만든다.

    * `ReactiveQueryByExampleExecutor<Employee> ` 를 상속하면 Query by Example을 사용할 수 있다.

    * ` <S extends T> Mono<S> findOne(Example<S> example);`  Example Type을 인자로 받은 메소드들이 생성되고 아래처럼 도메인 객체의 Example 객체를 생성한다.

```java
Employee e = new Employee();
e.setFirstName("Bilbo");
Example<Employee> example = Example.of(e);
```


* `Example` : QBE를 위한 오브젝트로 probe와 matcher로 이루어져 있는 데, probe는 조건으로 사용하려는 값들을 가지고 POJO 객체고 matcher는 프로브가 어떻게 사용될 것인지 제어하는 매처이다.

    * Example은 기본적으로 프로브에 들어가있는 값들을 통해 쿼리를 만들고 값들은 저장된 레코드들과 일치해야 한다.

    * ExampleMatcher를 생성해서 매쳐를 조절할 수도 있다.


```java
Employee e = new Employee();
e.setLastName("baggins"); // Lowercase lastName
ExampleMatcher matcher = ExampleMatcher.matching()
.withIgnoreCase()
.withMatcher("lastName", startsWith())
.withIncludeNullValues();
Example<Employee> example = Example.of(e, matcher);
```



## Querying with MongoOperations

* MongoTemplate을 이용해서 쿼리를 작성하는 방법
* 몽고디비에서만 사용할 수 있으므로 위처럼 작성할 수 있으면 사용하는게 가장 좋다.
* `ReactiveMongoTemplate` 를 대신 사용하면 Reactor type들을 이용할 수 있기 때문에 대신 사용하는 것이 좋다. `ReactiveMongoOperations` 를 이용해서 사용할 수 있다. 



## Logging reative operations

* Spring Boot는 확장 가능한 로깅을 지원해주는 데, `logback.xml` 을 생성해서 `src/main/resources` 에 두고 설정을 추가해서 쓸 수 있다. 
* `application.properties` 에 로그레벨을 설정할 수 있다.































