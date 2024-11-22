# XML
XML(eXtensible Markup Language)은 데이터를 저장하고 전송하기 위한 마크업 언어이다.  
마크업은 이름에서처럼 어떠한 표시를 통해서 텍스트를 구분하다는 것을 의미한다.  
XML은 마크업으로 태그를 사용한다.

## 구조
```xml
<library>
    <book>
        <title>XML Guide</title>
        <author>John Doe</author>
        <year>2024</year>
    </book>
    <book>
        <title>Learning Java</title>
        <author>Jane Smith</author>
        <year>2023</year>
    </book>
</library>
```
구조는 위와 같이 하나의 루트 태그를 가지고 하위 태그들을 계층적으로 표현한다.  
생김새는 HTML과 많이 닮아있다.

## HTML과 XML 비교
1. HTML의 목적은 데이터를 표현하는 데 있고, XML은 데이터를 저장하거나 전송하는 데 있다.
2. HTML은 미리 정해진 태그를 사용해야해서 마크업의 확장성이 없지만, XML은 사용자 정의 태그를 사용한다.
3. HTML은 태그에 대소문자를 구분하지 않지만, XML은 구분한다.

> 리액트에서 사용하는 JSX(JavaScriptXML) 태그를 사용자 정의가 가능해서 XML이라는 이름을 붙인 것 같다.

## JSON과 XML 비교

**json**
```json
{
  "person": {
    "name": "John",
    "age": 30,
    "hobbies": ["reading", "cycling"]
  }
}
```
**xml**
``` xml
<person>
  <name>John</name>
  <age>30</age>
  <hobbies>
    <hobby>reading</hobby>
    <hobby>cycling</hobby>
  </hobbies>
</person>

```

1. xml은 태그로 데이터를 구분하기 때문에 json에 비해서 더 많은 텍스트를 사용해야한다.
2. 태그가 많아질수록 xml은 json에 비해서 읽기가 힘들어질 수 있다.
3. json에 비해 xml을 파싱하는 데 더 많은 노력이 필요하다.


## XML 사용

### sitemap.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://www.example.com/</loc>
    <lastmod>2024-11-01</lastmod>
    <changefreq>daily</changefreq>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://www.example.com/about</loc>
    <lastmod>2024-10-20</lastmod>
    <changefreq>monthly</changefreq>
    <priority>0.8</priority>
  </url>
</urlset>
```
검색엔진에게 웹사이트에 어떤 페이지가 존재하는지 알려줘서 효율적으로 사이트를 크롤링하도록 도와준다.  
sitemap도 json으로 하면 안되나 싶긴 하지만 sitemap은 xml로만 제출이 가능하다.

### bean.xml
```xml
<beans>
    <bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource">
        <property name="url" value="jdbc:mysql://localhost:3306/mydb"/>
        <property name="username" value="root"/>
        <property name="password" value="password"/>
    </bean>
</beans>
```
스프링에서 bean 설정을 할 떄 사용한다.  
하지만 요즘에는 어노테이션 기반으로 설정해서 거의 사용하지 않는다.

### rss.xml
개츠비로 블로그 사용하려고 할 때 봤었다.  
처음에 용도는 몰랐는데 사용자가 RSS 피드를 구독하면, 새로운 글이 게시될 때마다 해당 콘텐츠를 자동으로 받을 수 있다고 한다.  
근데 아직 잘 모르겠다..
