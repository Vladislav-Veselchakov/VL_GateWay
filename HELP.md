# Getting Started

### Reference Documentation

For further reference, please consider the following sections:

* [Official Apache Maven documentation](https://maven.apache.org/guides/index.html)
* [Spring Boot Maven Plugin Reference Guide](https://docs.spring.io/spring-boot/docs/3.0.5/maven-plugin/reference/html/)
* [Create an OCI image](https://docs.spring.io/spring-boot/docs/3.0.5/maven-plugin/reference/html/#build-image)
* [Gateway](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/)
* [Spring Boot DevTools](https://docs.spring.io/spring-boot/docs/3.0.5/reference/htmlsingle/#using.devtools)
* [Eureka Discovery Client](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/#service-discovery-eureka-clients)

### Guides

The following guides illustrate how to use some features concretely:

* [Using Spring Cloud Gateway](https://github.com/spring-cloud-samples/spring-cloud-gateway-sample)
* [Service Registration and Discovery with Eureka and Spring Cloud](https://spring.io/guides/gs/service-registration-and-discovery/)

ВЛ: сделано с сайта
[https://sysout.ru/spring-cloud-api-gateway/](https://sysout.ru/spring-cloud-api-gateway/)

Spring Cloud API Gateway
В этой статье продолжим дорабатывать предыдущий пример с Eureka и client-side load balancing —  добавим в него Spring Cloud API Gateway.

Что такое Spring Cloud API Gateway
Это отдельное Spring Boot приложение, через которое проходят все запросы, реализация шаблона Reverse Proxy. То есть микросервисы не знают друг о друге, а обращаются к прокси. Внешнему пользователю тоже известен только прокси. Прокси, в свою очередь, анализирует запрос, перенаправляет его к нужному микросервису и возвращает ответ обратно. Ниже мы рассмотрим, как в прокси задать условия — какой запрос к какому микросервису направить.

Зачем нужен Spring Cloud API Gateway
Например для того, чтобы зафиксировать REST API. Представьте, что вы разрабатываете микросервисы и обсуждаете с фронтенд-разработчиком REST API. У вас постоянно что-то меняется: сегодня микросервис выдает данные по такому url, завтра — по-другому. А то и вовсе данные будут выдаваться новым микросервисом.

Можно зафиксировать REST API на прокси и менять внутреннюю структуру как угодно. Снаружи ничего не поменяется, просто прокси будет обращаться по другим адресам. А обращения к самому прокси останутся прежними.

Spring Cloud API Gateway vs. Zuul
Пример сделан на новом Spring Cloud API Gateway.

Zuul имеет примерно ту же функциональность, но Zuul 1.x не реактивный. Zuul 2.0 реактивный, но Spring его поддерживает хуже, чем Zuul 1.x. Поэтому если у нас реактивный стек, то Spring Cloud API Gateway — лучший выбор.

Наша структура
Мы продолжим разрабатывать предыдущий пример. Есть микросервис Zoo — он выдает случайное животное. Раньше пользователь в браузере обращался к нему, но теперь Zoo будет запущен в двух экземплярах (вот, еще одно преимущество), а пользователь будет обращаться к прокси, запущенному на порту 8080. Это прокси и есть наш Spring Cloud API Gateway.

Есть также микросервис Random Animal — к нему обращался Zoo, чтобы получить это животное, но теперь Zoo будет обращаться тоже к прокси. А прокси, в свою очередь, к Random Animal.  Random Animal тоже запущен в двух экземплярах (но можно запустить сколько угодно, пример будет работать).

В общем картина такая:

Запросы проходят через прокси
Запросы проходят через прокси
Proxy должен как-то решать, к какому микросервису направлять пришедший запрос.

Например, пользователь в браузере обращается к прокси с запросом localhost:8080/zoo/animals/any, прокси решает, к какому микросервису перенаправить запрос, и выбирает Zoo. Потом Zoo обращается к прокси с другим запросом, и прокси решает перенаправить его к Random-Animal.
Эта логика задается с помощью элементов, перечисленных ниже.

Элементы Spring Cloud API Gateway
(Spring Cloud API Gateway работает на сервере Netty.)

Spring Cloud API Gateway
Spring Cloud API Gateway
Чтобы сопоставить входной url (идущий в Spring Cloud Gateway) выходному url (идущему к микросервису), нужно задать три пункта:

Predicate: условие, при котором запрос перенаправляется (например, если url соответствует такому-то шаблону).
URI: содержит uri куда перенаправляем запрос (к какому микросервису).
Filter (необязательно): как модифицировать запрос (на пути туда или обратно).
Эти три пункта (еще идентификатор Route-а) объединены в Route — основной строительный блок. Из нескольких таких блоков и состоит настройка.

Два Route-а мы настроим ниже. Но сначала добавим в приложение Maven-зависимость и зададим ему имя (для Eureka).

Maven-зависимость
Чтобы сделать Spring Boot приложение API Gateway-ем, добавим зависимость:

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
Также (поскольку мы уже включили в предыдущий пример Eureka), чтобы другие микросервисы могли обращаться к API Gateway-ю по имени, а не по адресу с портом (типа localhost:8080), сделаем Spring Cloud API Gateway клиентом Eureka:

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
А обращаться они будут к нему по имени proxy, что и зададим ниже.

Имя приложения API Gateway
В нашей системе API Gateway будет обнаруживаться по имени proxy — так мы назовем наше приложение:

server:
port: 8080
spring:
application:
name: proxy
А запущен он будет на порту 8080.

В примере обращаться к прокси по имени proxy будет микросервис Zoo. (Пользователь в браузере для обращения к прокси вводит обычный адрес localhost:8080/…, где запущен прокси).
Сопоставление адресов: настройка Route-ов: URI, Predicate и Filter
Итак, зададим, что если url обращения к прокси (который у нас на порту 8080) начинается с /zoo:

localhost:8080/zoo/....
то прокси переводит обращение на микросервис Zoo.

А если с /random-animal:

localhost:8080/random-animal/...
то на микросервис Random Animal:

Какие запросы на какой микросервис идут
Какие запросы на какой микросервис идут
Еще раз обращаю внимание, что адрес до первого слеша / может быть как localhost:8080, так и просто имя proxy. Это зависит от того, кто обращается. Микросервисы обращаются по имени proxy благодаря Eureka, из браузера так нельзя.
Вообще настроить Spring Cloud API Gateway можно как в коде, так и в application.yml.

Настройка в коде
Видно, что в настройке фигурируют два Route (как на картинке выше).  Для каждого задан Uri, Predicate и Filter:

@Configuration
class ProxyConfig {
@Bean
RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
return builder.routes()
.route("random_animal_route",
route -> route.path("/random-animal/**")
.and()
.method(HttpMethod.GET)
.filters(filter -> filter.stripPrefix(1)
)
.uri("lb://random-animal"))
.route("zoo_route",
route -> route.path("/zoo/**")
.filters(filter -> filter.stripPrefix(1)
)
.uri("lb://zoo"))
.build();
}
}
Uri
В uri() задан микросервис, куда идет перенаправление: zoo или random-animal. Важно не пытаться прописать тут вложенные пути — не сработает. Только имя микросервиса (под которым он зарегистрирован в Eureka). Префикс lb говорит о том, что обращение к микросервису должно проходить через балансировщик нагрузки — то есть заодно еще автоматически будет принято решение о том, к какому именно экземпляру микросервиса обратиться.

Predicate
Здесь могут быть заданы любые условия, по которым запрос отбирается. Как уже говорилось, мы отбираем запрос по path() — задаем шаблон «url начинается с того-то». Еще сказано, что это должен быть метод GET (просто для демонстрации возможностей). Все эти заданные условия и есть предикат.

Filter
В фильтре мы отбрасываем вот эту начальную часть url, по которой выбрали запрос (/zoo/ и /random-animal/). В контроллер микросервиса запрос пойдет без этой части.

Аргумент 1 в методе:

filter.stripPrefix(1)
означает, что именно одну часть отбрасываем.  Если бы мы отбирали запрос по двум начальным частям, например localhost:8080/zoo/part2/..., то отбросили бы две части, чтобы не тянуть их в контроллер.

Проверка
Контроллер в микросервисе Zoo у нас такой:

@RestController
public class ZooController {
@Autowired
private RandomAnimalClient randomAnimalClient;
@GetMapping("/animals/any")
ResponseEntity<Animal> seeAnyAnimal(){
return randomAnimalClient.random();
}
}
Обращение к нему через прокси будет таким:

Обращение через прокси
Обращение через прокси
/zoo отбрасывается, в контроллер идет /animals/any.

А внутри контроллера в Zoo обращение ко второму микросервису уже по имени:

@Component
public class RandomAnimalClient {
private static final Logger LOGGER = LoggerFactory
.getLogger(RandomAnimalClient.class);
private final RestTemplate loadBalancedTemplate;
RandomAnimalClient(@LoadBalanced RestTemplate loadBalancedTemplate,
DiscoveryClient discoveryClient) {
this.loadBalancedTemplate = loadBalancedTemplate;
}
public ResponseEntity<Animal> random1() {
LOGGER.debug("Sending  request for animal {}");
return loadBalancedTemplate.getForEntity("http://proxy/random-animal/random",
Animal.class);
}
}
В запросе

http://proxy/random-animal/random
использовано имя нашего API Gateway в Eureka — proxy. Обращение сделано через @LoadBalanced RestTemplate — это значит, что экземпляров proxy вообще может быть несколько (можно запустить API Gateway на нескольких портах — и запрос будет работать).

Часть url /random-animal/ служит для того, чтобы отобрать запрос для направления в микросервис Random Animal, а затем отбрасывается. В микросервис идет только /random.

Соответственно  контроллер в микросервисе Random Animal принимает запросы, начинающиеся с /random:

@RestController
public class RandomAnimalController {
private final AnimalDao animalDao;
public RandomAnimalController(AnimalDao animalDao){
this.animalDao=animalDao;
}
@GetMapping("/random")
public Animal randomAnimal(){
Animal animal=animalDao.random();
System.out.println(animal);
return animal;
}
}
Настройка в application.yml
Настройку из кода можно перенести в файл настроек:

spring:
cloud:
gateway:
routes:
- id: random_animal_route
uri: lb://random-animal
predicates:
- Path=/random-animal/**
filters:
- StripPrefix=1
- id: zoo_route
uri: lb://zoo
predicates:
- Path=/zoo/**
filters:
- StripPrefix=1
Итоги
Пример можно скачать на GitHub.
- [https://github.com/myluckagain/sysout/tree/master/cloud2](https://github.com/myluckagain/sysout/tree/master/cloud2)
- Все три части — Zoo, Random Animal и Proxy — можно запускать на любом количестве портов.

Далее рассмотрим Spring Cloud Configuration Server.