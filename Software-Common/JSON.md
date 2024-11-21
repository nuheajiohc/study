# JSON
- JSON(JavaScript Object Notation)은 자바스크립트 객체 형식으로 데이터를 교환하는 문자열이다.  
- java, javascript, python 등의 언어에서 데이터를 교환할 때 쓰인다.

## 형식
이름에서와 같이 자바스크립트의 객체 문법을 가져와서 적용했다.  
하지만 객체 형식만 가능한 것은 아니고, 배열도 가능하고 문자열도 가능하다.

### 객체 형식
``` json
{
    "name" : "gildong",
    "hobby" : {
        "운동" : ["헬스", "런닝"],
        "음악" : ["기타", "노래"]
    }
}
```

위와 같이 키:값 형식의 객체로 표현할 수 있다.  
일반적으로 위와 같은 형태로 데이터를 전송한다.

### 배열 형식
```json
["apple", "banana", "cherry"]
```
위와 같이 배열 형태로도 표현이 가능하다.   

### 문자열
```json
"Hello, world!"
```
단순하게 문자열로도 보낼 수 있다.


## 타입
자바스크립트의 문법을 가져와서 만든 형식이지만 undefined는 사용할 수 없다.  
아래와 같은 타입만 가능하다.
- number
- string
- object
- array
- true
- false
- null

## 직렬화(객체 -> JSON)
직렬화란 어떤 한 시스템에서 만든 데이터 집합을 다른 시스템으로 전송하기 위해 문자열로 변환하는 과정을 말한다.  
각 시스템 사이에는 사용하는 문법이 다르기 때문에 데이터를 전달해줄 매개체가 필요한데 여기서는 그 역할을 JSON이 해준다.  
여기서는 객체를 JSON으로 바꾼다고 했지만 타입 부분에서 말했듯 명시한 타입들은 JSON으로 바꿀 수 있다.
> 나는 어떤 형식의 데이터를 문자열로 쭈욱 펴내서 전달한다고 이해했다.. 연속적인 형태... 문자열.. 이런느낌으로 

### 자바스크립트에서 직렬화
`JSON.stringify(직렬화 할 데이터)` 를 사용해서 직렬화를 한다.  
그러면 문자로 직렬화가 된다.
``` javascript
const data = {"name" : "gildong"};
JSON.stringify(data);

//결과
'{"name":"gildong"}'
```

### 자바에서 직렬화
`ObjectMapper` 클래스를 사용하여 직렬화를 진행한다.  
jackson 라이브러리를 dependency에 추가해야한다.
``` java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```
```java
import com.fasterxml.jackson.databind.ObjectMapper;

public class Main {
    public static void main(String[] args) {
        // Person 객체 생성
        Person person = new Person("Alice", 30);

        // ObjectMapper 생성
        ObjectMapper objectMapper = new ObjectMapper();

        try {
            // 객체를 JSON 문자열로 직렬화
            String jsonString = objectMapper.writeValueAsString(person);
            System.out.println(jsonString);  // 출력: {"name":"Alice","age":30}
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

스프링 부트를 사용하면 `@ResponseBody` 통해 자동으로 직렬화가 이루어진다. 
```java
@RestController
public class PersonController {

    @PostMapping("/person")
    public Person createPerson(@RequestBody Person person) {
        // JSON -> Java 객체 (역직렬화)
        System.out.println(person.getName());  // 요청 본문에서 "name" 필드 값 출력
        return person;  // 객체 -> JSON (직렬화)
    }
}
```


## 역직렬화(JSON -> 객체)
JSON을 시스템의 데이터로 파싱하는 과정을 말한다.

### 자바스크립트에서 역직렬화
`JSON.parse(데이터)`를 사용해서 역직렬화를 한다.
```javascript
// JSON 문자열
const json = '{"name": "Alice", "age": 30, "hobbies": ["reading", "traveling"]}';

// JSON.parse()를 사용하여 JSON 문자열을 객체로 역직렬화
const person = JSON.parse(jsonString);

console.log(person); // 출력: { name: 'Alice', age: 30, hobbies: [ 'reading', 'traveling' ] }
console.log(person.name); // 출력: Alice
console.log(person.hobbies); // 출력: [ 'reading', 'traveling' ]
console.log(person.hobbies[0]); // 출력: reading

```
### 자바에서 역직렬화
```java
import com.fasterxml.jackson.databind.ObjectMapper;

public class Main {
    public static void main(String[] args) {
        // JSON 문자열
        String jsonString = "{\"name\": \"Alice\", \"age\": 30}";

        // ObjectMapper 생성
        ObjectMapper objectMapper = new ObjectMapper();

        try {
            // JSON 문자열을 Java 객체로 역직렬화
            Person person = objectMapper.readValue(jsonString, Person.class);

            // 결과 출력
            System.out.println(person);
            System.out.println("Name: " + person.getName());
            System.out.println("Age: " + person.getAge());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
/*
결과
Person{name='Alice', age=30}
Name: Alice
Age: 30

 */
```
스프링부트에서는 `@RequestBody`를 통해 자동으로 역직렬화가 이루어진다.

## 주의할 점
- 역직렬화할 때 문자열은 작은 따옴표가 아닌 큰 따옴표를 사용해야한다.
- 객체나 배열 안의 문자열도 큰 따옴표를 사용해야 한다.
```javascript
console.log(JSON.parse('42'));  // 출력: 42 (숫자)
console.log(typeof JSON.parse('42'));  // 출력: "number"

console.log(JSON.parse('"42"'));  // 출력: 42 (숫자처럼 보이는 문자열)
console.log(typeof JSON.parse('"42"'));  // 출력: "string"

console.log(JSON.parse('"hello"'));  // 출력: hello (문자열)
console.log(typeof JSON.parse('"hello"'));  // 출력: "string"

console.log(JSON.parse('hello'));  // 오류 발생
// SyntaxError: Unexpected token h in JSON at position 0


const str1 = '"true"';
console.log(JSON.parse(str1)); // 출력: "true" (문자열로 반환됨)

const str2 = true;
console.log(JSON.parse(str2)); // 출력: "true (boolean으로 반환됨)

const a = 1
const b = "1"
const c = '"1"'
console.log(typeof JSON.parse(a)) // number으로 반환
console.log(typeof JSON.parse(b)) // number으로 반환
console.log(typeof JSON.parse(c)) // string으로 반환
```
## 내 생각
다양한 형식으로 전송이 가능하지만 JSON이라는 이름에 맞게 객체 형식으로 보내는 것이 좋은 것 같다.  
프론트엔드와 백엔드를 분리하여 개발할 때 JSON을 통해 주로 데이터를 전송하게 되는데, 상황에 따라 형식을 달리해서 데이터 전송을 하면 데이터 받는 쪽에서 번거로울 수 있다고 생각하기 떄문이다.  

그러므로 배열을 응답해야 하는 경우라도 응답 구조를 아래와 객체로 감싸서 일관된 형식으로 전송하는 것이 좋은 것 같다.
- 비추천 하는 형식
```json
[
        { "id": 1, "name": "Alice" },
        { "id": 2, "name": "Bob" },
        { "id": 3, "name": "Charlie" }
]
```

- 추천 하는 형식
```json
{
    "status": "success",
    "message": "데이터가 정상적으로 처리되었습니다.",
    "data": [
        { "id": 1, "name": "Alice" },
        { "id": 2, "name": "Bob" },
        { "id": 3, "name": "Charlie" }
    ]
}

```

이처럼 일관된 형식을 통해 데이터와 메타 데이터를 함께 보내주는 것이 응답 처리하는 입장에서도 편리할 것이라고 생각한다. 