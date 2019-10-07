---
layout: post
title:  Spring5_Chapter3_SpringMVC
date:   2019-04-14
author: ag-lee
categories: Spring5Study
---



# 3장. 스프링 MVC

## 레시피 3-1 간단한 스프링 MVC 웹 애플리케이션 개발하기

간단한 웹 애플리케이션을 스프링MVC로 개발하면서 프레임워크의 기본 개념과 구성 방법

### 프론트 컨트롤러 

스프링의 중심 컨트롤러로 MVC에서는 __dispatcher serlvet__을 의미

- 모든 웹요청은 dispatcher servlet이 받아 가장 먼저 Controller에 전달한다.
- 컨트롤러는 `@Controller` 나 `@RestController` 를 붙여서 처리한다.
- `@Controller` 에 요청이 들어오면 스프링은 `@RequestMapping` 이 붙은 핸들러메소드를 찾아 매핑한다.
- `@RequestMapping` 을 붙인 핸들러 메소드의 Arguments
    - HttpServletRequest, HttpServletResponse
    - 매개변수 (`@RequestParam`)
    - 모델 속성 (`@ModelAttribute`)
    - 요청에 포함된 쿠키 값 (`@CookieValue`)
    - 모델에 속성을 추가하기 위해 사용하는 Map, ModelMap
    - 객체 바인딩/유효성을 검증한 결과를 가져올 때 Errors, BindingResult
        - 주로 Controller에서 Validation을 한 후 결과를 담아서 View로 전달하는 역할
        - 결과를 담아서 내보냈을 때 어떻게 나오지?
    - 세션 처리를 완료했음을 알릴 때 사용하는 SessionStatus
- Controller는 뷰에 해당하는 String값을 반환하는게 보통이지만, 반환값을 void로 선언하면 핸들러 메소드나 컨트롤러 이름에 따라 기본적인 뷰가 자동으로 결정된다.



- RequestMapping을 세분화해 `@GetMapping` & `@PostMapping` 으로 쓸 수 있다.



### 컨트롤러 코드

```java
@Controller
@RequestMapping("/reservattionQuery")
public class ReservationQueryController {
    private final ReservationService reservationService;
    
    public ReservationQueryController(ReservationService reservationService) {
        this.reservationService = reservationService;
    }
    
    @GetMapping
    public void setupForm() {
        
    }
    
    @PostMapping
    public void submitForm(@RequestParam("courtName") String courName, Model model) {
        List<Reservation> reservations = Collections.emptyList();
        if(courtName != null) {
            reservations = reservationService.query(courtName);
        }
        model.addAttribute("reservations", reservations);
        return "reservationQuery";
	}
}
```

- `/reservationQuery` 라는 요청이 GET메소드로 들어왔을 때, setupForm() 핸들러메소드가 실행되는데 이 함수에서는 아무 동작도 하지 않지만, 요청URL로 뷰가 mapping되므로 reservationQuery라는 이름의 view가 반환된다.



## 레시피 3-2 @RequestMapping에서 요청 매핑하기

`@Controller` 클래스의 핸들러 메소드에 선언된 다양한 `@RequestMapping` 설정으로 요청을 매핑하는 전략 정의



### 1. 메소드에 따라 요청 매핑하기

핸들러 메서드에 `@RequestMapping` 직접 붙여 URL 패턴을 기재한다.

```java
@Controller
public class MemberController {
    private MemberService memberService;
    
    public MemberController(MemberService memberService) {
        this.memberService = memberService;
    }
    
    @RequestMapping("/member/add")
    public String addMember(Model model) {
        model.addAttribute("member", new Member());
        model.addAttribute("guests", memberService.list());
        return "memberList";
    }
    
    @RequestMapping(value={"/member/remove", "member/delete"}, method = 																			RequestMethod.GET)
    public String removeMember(@RequestParam("memberName")String memberName) {
        memberService.remove(memberName);
        return "redirect:";
    }
}
```



### 2. 클래스에 따라 요청 매핑하기

- `@RequestMapping` 을 컨트롤러 클래스 자체에 붙여서 매핑한다.
- 클래스 레벨에 `@RequestMapping` 을 붙이면 클래스 내의 전체메소드에 일일이 붙이지 않아도 된다.
- URL을 폭넓게 매치하기 위해 (*)와일드카드를 이용할 수 있다.

```java
@Controller
@RequestMapping("/member/*")
public class MemberController {
    private final MemberService memberService;
    
    public MemberController(MemberService memberService) {
        this.memberService = memberService;
    }
    
    @RequestMapping("add")
    public String addMember(Model model) {
        model.addAttribute("member", new Member());
        model.addAttribute("guests", memberService.list());
        return "memberList";
    }
    
    @RequestMapping("display/{member}")
    public String displayMember(@PathVariable("member") String member, Model model) {
        model.addAttribute("member", memberService.find(member).orElse(null));
        return "member";
    }
    
    @RequestMapping
    public void memberList() {}
}
```

- `/member/` 로 시작하는 모든 요청은 이 컨트롤러의 핸들러 메소드 중 하나로 연결된다.
- `@RequestMapping` 를 가진 메소드는 `/member/` 로 시작하는 요청 중, 다른 요청에 걸리지 않은 모든 요청이 이 메소드를 실행한다. 반환값은 void이므로 자신의 메소드이름과 같은 view로 요청을 넘긴다.



### 3. HTTP 요청 메소드에 따라 요청 매핑하기

- `@RequestMapping` 내부에 HTTP 요청 메소드를 명시해 매핑하는 방법
- SpringMVC는 가장 널리 쓰는 요청을 편리하게 사용하도록 어노테이션을 지원한다.
    - `@PostMapping`, `@GetMapping`, `@DeleteMapping`, `@PutMapping`



## 레시피 3-3 핸들러 인터셉터로 요청 가로채기

스프링 MVC핸들러로 웹 요청을 넘기기 전후에 __핸들러 인터셉터__를 통해 전처리, 후처리를 통해 뷰로 모델을 전달하기 전 조작한다.

- 핸들러 인터셉터는 특정 URL에만 적용되도록 매핑할 수 있다.

- 웹 애플리케이션 컨텍스트에 구성되므로 내부에 사용된 모든 빈을 참조할 수 있다.

- `HandlerInterceptor` 인터페이스를 구현한다.

    - `preHandle()`, `postHandle()` 메소드는 핸들러가 요청을 처리하기 직전과 직후에 호출
    - `postHandle()` 은 핸들러가 반환한 ModelAndView객체에 접근이 가능해 내부의 속성을 꺼내 조작이 가능하다.
    - `afterCompletion()` 은 요청 처리가 모두 끝난 후 (뷰 렌더링까지 완료) 이후 호출

- `HandleInterceptor` 인터페이스를 이용하면 세 함수를 모두 구현해야 하지만, 인터셉터 어댑터 클래스를 상속받아 사용하면 필요한 메소드만 오버라이드해서 사용할 수 있다.

    - `HandleInterceptorAdapter`
    - 인터셉터는  `WebMvcConfigurer` 인터페이스를 구현한 구성 클래스 InterceptionConfiguration을 만들어 `addInterceptor()`를 오버라이드해 만들 수 있다.
        - 특정 URL에만 인터페이스를 적용하고 싶을 땐 `addPatterns("url")` 을 이용한다.
        - 특정 URL을 제외하고 적용하고 싶을 땐 `excludePathPatterns("url")` 을 이용한다.

```java
@Configuration
public class InterceptionConfiguration implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(aInterceptor());
        registry.addInterceptor(bInterceptor())
                    .addPathPatterns("/b");
        registry.addInterceptor(cInterceptor())
                    .excludePathPatterns("/c");
    }
    
    @Bean
    public AInterceptor aInterceptor() {
        return new AInterceptor();
    }
    
    @Bean
    public BInterceptor bInterceptor() {
        return new BInterceptor();
    }
    
    @Bean
    public CInterceptor cInterceptor() {
        return new CInterceptor();
    }
}
```

    

## 레시피 3-4 유저 로케일 해석하기

다국어를 지원하는 웹 애플리케이션에서 각 유저마다 선호하는 로케일을 식별하고 그에 알맞는 콘텐츠를 표시한다.

- Spring MVC은 `LocaleResolver` 인터페이스를 구현한 로케일 리졸버가 유저 로케일을 식별한다.
    - 따로 인터페이스를 구현해 사용할 수도 있다.
- `DispatcherServlet` 당 하나의 로케일 리졸버만 빈으로 등록이 가능하다.
- 자동감지를 위해서는 localeResolver로 빈을 명명한다.



### HTTP 요청 헤더에 따라 로케일 해석하기

- `AcceptHeaderLocaleResolver` 는 스프링의 기본 로케일 리졸버이다. 
    - `accept-language` 요청 헤더 값에 따라 로케일을 해석한다.
    - 웹 브라우저는 자신을 실행한 os의 로케일 설정으로 이 헤더를 설정한다.



### 세션 속성에 따라 로케일 해석하기

- `SessionLocaleResolver` 는 유저 세션에 사전 정의된 속성에 따라 로케일을 해석한다.

    - 세션정보가 없으면 `accept-language` 요청 헤더 값에 따라 로케일을 해석한다.
    - 세션 정보가 없는 경우, `setDefaultLocale()`로 기본 값을 정의해줄 수 있다.

```java
@Bean
public LocaleResolver localeResolver() {
    SessionLocaleResolver localeResolver = new SessionLocaleResolver();
    localeResolver.setDefaultLocale(new Locale("en"));
    return localeResolver;
}
```

    

### 쿠키에 따라 로케일 해석하기

- `CookieLocaleResolver` 는 브라우저의 쿠키값에 따라 로케일을 해석한다.
    - 쿠키정보가 없으면 `accept-language` 요청 헤더 값에 따라 로케일을 해석한다.
    - 쿠키 정보가 없는 경우, `setDefaultLocale()`로 기본 값을 정의해줄 수 있다.
    - 쿠키 설정은 `setCookieName()` 과 `setCookieMaxAge()`로 커스터마이징할 수 있다.
    - 저장된 쿠키값을 변경해 유저로케일을 변경할 수 있다.



### 유저 로케일 변경하기

- `LocaleChangeInterceptor` 를 핸들러 매핑에 적용해 변경할 수 있다.

    - HTTP 요청에 특정한 매개변수가 없는지 감지해 인터셉터의 paramName프로퍼티값으로 지정해 그 값으로 유저 로케일을 변경할 수 있다.
    - 아래 코드인 경우, 아래 url처럼 접속해 로케일을 변경할 수 있다.
        - localhost:8080/?language=en_US
        - localhost:8080/?language=de

```java
@Configuration
public class I18NConfiguration implements WebMvcConfiguerer {
    @Override
    public void addInterceptor(InterceptorRegistry registry) {
        registry.addInterceptor(localeInterceptor());
    }
    
    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor(){
        LocaleChangeInterceptor localeChangeInterceptor = new LocaleChangeInterceptor();
        localeChangeInterceptor.setParamName("language");
        return localeChangeInterceptor;
    }
}
```



## 레시피 3-5 로케일별 텍스트 메세지 외부화하기

다국어를 지원하는 웹 애플리케이션은 유저가 원하는 로케일로 웹 페이지를 보여줘야한다.

- 로케일마다 페이지를 따로 두지 않으려면 로케일 관련 텍스트 메세지를 외부화해 웹 페이지를 로케일에 독립적으로 개발한다.
- 스프링은 `MessageSource` 인터페이스를 구현한 메세지 소스로 텍스트 메세지를 해석가능하다.
- \<spring:message\> 태그를 사용하면 원하는 코드에 맞게 해석된 메세지가 출력된다.



1. `MessageSource` 형 빈을 등록한다. 

    - messageSource라고 명명하면 dispatcher servlet이 자동으로 찾는다. dispatcher servlet 당 하나의 메세지 소스만 등록이 가능하다.

    - `ResourceBundleMessageSource` 구현체는 로케일마다 따로 배치한 리소스 번들을 이용해 메세지를 해석한다.

        - WebConfiguration에 구현하면 basename이 messages인 리소스 번들을 로드한다.

```java
@Bean
public MessageSource messageSource() {
    ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
    mssageSource.setBasename("messages");
    return mssageSource;
}
```



2. 로케일 메세지를 담아둘 리소스번들 `messages.porperties` 와 `messages_de.properties` 를 생성해 각각 클래스 패스 루트에 둔다.

```properties
# default
welcome.title=Welcome
welcome.message=Welcome to Court Reservation System

# de
welcome.title=Willkommen
welcome.message=Willkommen zum Spielplatz-Reservierungssytem
```

    

3. welcome.jsp 에서 \<spring:message\> 태그로 주어진 코드에 해당하는 메세지를 보여준다.

```jsp
<%@ tablib prefix="spring" uri="http://www.springframework.org/tabs" %>

<html>
    <head>
        <title><spring:message code="welcome.title" text="WelCome" /></title>
    </head>
    <body>
        <h2>
            <spring:message code="welcome.message" text="Welcome to Court Reservation System" />
        </h2>
        
    </body>
</html>

```



## 레시피 3-6 이름으로 뷰 해석하기

핸들러가 요청 처리를 마치고 뷰 이름을 반환하면 DispatcherServlet은 논리 뷰 이름에 따라 뷰를 해석한다.

- `viewResolver` 를 이용해 뷰를 해석할 수 있도록 한다.



### 템플릿명과 위치에 따라 뷰 해석하기

기본으로 템플릿의 이름과 위치에 뷰를 직접 매핑할 수 있다.

- `InternalResourceViewResolver`는  prefix/suffix를 이용해 뷰 이름을 특정 애플리케이션 디렉터리에 대응시킨다.

```java
@Bean
public InternalResourceViewResolver viewResolver() {
    InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
    viewResolver.setPrefix("/WEB-INF/jsp/");
    viewResolver.setSuffix(".jsp");
    return viewResolver;
}

```

    

### XML 구성 파일에 따라 뷰 해석하기

view를 bean으로 선언하고 해당 빈 이름으로 해석한다.

- `application-context.xml` 에 선언해도 되지만, 별도 파일로 빼는게 좋다.
- Configuration 파일에서 xml 파일을 로드해야한다. (p192)

```xml
<bean id="welcome"
      class="org.springframework.web.servlet.JstlView">
      <property name="url" value="/WEB-INF/jsp/welcome.jsp"></property>
</bean>

<bean id="reservationQuery"
      class="org.springframework.web.servlet.JstlView">
      <property name="url" value="/WEB-INF/jsp/reservationQuery.jsp"></property>
</bean>

```



### 리소스 번들에 따라 뷰 해석하기

p193



### 여러 리졸버를 이용해 뷰 해석하기

P193



## 레시피 3-7 뷰와 콘텐트 협상 활용하기

컨트롤러에서 확장자가 없는 URL을 매핑해 어떤 요청이 들어와도 올바른 콘텐트 타입을 반환하는 전략을 수립한다.

- 클라이언트가 보낸 요청에는 프레임워크가 클라이언트에게 반환할 콘텐트 및 타입을 파악하는 데 필요한 프로퍼티가 포함되어 있다.
- 확장자와 HTTP Accept헤더를 통해 판단하는데, 확장자가 없는 경우에는 HTTP Accept 헤더를 들춰보면 적절한 뷰 타입을 결정할 수 있다.
- Spring MVC가 지원하는 ContentNegotiatingViewResolver를 통해 헤더를 살펴본다.



### ContentNegotiatingViewResolver

1. 경로에 포함된 확장자를 ContentNigotiatingManager 빈 구성 시 지정한 mediaTypes 맵을 이용해 기본 미디어 타입과 비교한다.
    - 요청경로에 확장자가 있지만 기본 미디어 타입과 매칭되는 확장자가 없으면 JAF의 FileTypeMap을 이용해 확장자의 미디어타입을 지정한다.
    - 요청경로에 확장자가 없으면 HTTP Accept  헤더를 활용해 지정된 값을 확인한다.
2. 이 정보를 바탕으로 나머지 리졸버를 순회하며 반환된 뷰에 가장 적합한 뷰를 결정한다.



## 레시피 3-8 뷰에 예외 매핑하기

예외가 발생해도 stack trace가 아닌 예외처리용 뷰를 보이도록한다.

- `HandlerExceptionResolver` 를 구현한 예외처리 리졸버 빈은 dispatcher servlet이 자동으로 감지한다.
- Spring MVC에 내장된 단순 예외 리졸버를 이용하면 예외 카테고리 별로 뷰를 하나씩 매핑할 수 있다.



### SimpleMappingExceptionResolver로 예외 매핑하기

```java
@Configuration
public class ExceptionHandlerConfiguration implements WebMvcConfigurer {
    @Override
    public void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> 																		exceptionResolvers)  {
        Properties exceptionMapping = new Properties();
        exceptionMapping.setProperty(ReservationAvailableException.class.getName(),																	"reservationNotAvailable");
        
        SimpleMappingExceptionResolver exceptionResolver = 
            							new SimpleMappingExceptionResolver();
        exceptionResolver.setExceptionMappings(exceptionMapping);
        exceptionResolver.setDefaultErrorView("error");
        return exceptionResolver;
        
    }
}

```

- ReservationAvailableException 예외 클래스를 reservationNotAvailable 논리 뷰에 매핑했다.
- 이와같이 발생한 예외 클래스 형에 따른 뷰를 유저에게 보여준다.
- 에러 페이지에서 자세한 오류 내용을 유저에게 보여주려면 ${exception} 변수를 사용해 인스턴스에 접근한다.



### `@ExceptionHandler`로 예외 매핑하기

`@ExceptionHandler` 를 븉여 예외 핸들러를 매핑한다. 작동원리는 `@RequestMapping` 과 비슷하다.

```java
@Controller
@RequestMapping("/reservationForm")
@SessionAttributes("reservation")
public class ReservationFormController {
	@ExceptionHandler(ReservationNotAvailableException.class)
    public String handle(ReservationAvailableException ex) {
        return "reservationNotAvailable";
    }
    
    @ExceptionHandler
    public String handleDefualt(Exception e) {
        return "error";
    }
}

```

- `@ExceptionHandler`

    - 렌더링할 뷰 이름, ModelAndView, View 등 여러 타입을 반환할 수 있다.
    - 자신을 둘러 싼 컨트롤러 안에서만 작동하기 때문에 다른 컨트롤러에서 예외가 발생하면 호출되지 않는다.

- 범용적인 예외처리 메서드는 별도 클래스로 빼내어 클래서 레벨에 `@ControllerAdvice` 를 붙인다.

    - application context에 존재하는 모든 컨트롤러에 적용된다.

```java
@ControllerAdvice
public class ExceptionHandlingAdvice {
    @ExceptionHandler(ReservationNotAvailableException.class)
    public String handle(ReservationAvailableException ex) {
        return "reservationNotAvailable";
    }
    
    @ExceptionHandler
    public String handleDefualt(Exception e) {
        return "error";
    }
}

```



## 레시피 3-9 컨트롤러에서 폼 처리하기

- 폼을 HTTP GET 요청하면 컨트롤러는 폼 뷰를 렌더링해 화면에 표시한다.
- 폼을 HTTP POST 요청하면 폼 데이터를 검증 후 요건에 맞게 처리한다.



### 폼 컨트롤러 작성하기

```java
@Controller
@RequestMapping("/reservationForm")
@SessionAttribute("reservation")
public class ReservationController {
	private ReservationService reservationService;
    
    @Autowired
    public ReservationService(ReservationService reservationService) {
        this.reservationService = reservationService;
    }
    
    @RequestMapping(method=RequestMethod.GET)
    public String setupForm(Mode model) {
        Reservation reservation = new Reservation();
        model.addAttribute("reservation", reservation);
        return "reservationForm";
    }
    
    @RequestMapping(method=RequestMethod.POST)
    public String submitForm(@ModelAttribute("reservation") Reservation reservation,
                            BindingResult result, SessionStatus status) {
        reservationSerivce.make(reservation);
        return "redirect:reservationSucess";
    }
}

```

- Model의 reservation에서 Reservation객체로 넣어줬다고 간주한다.
- `@SessionAttribute("reservation")` 은 필드를 세션에 보관했다가 폼을 여러차례 재전송해도 동일한 레퍼런스를 참조해 필요한 데이터를 가져오기 위한 것이다.  HTTP GET 핸들러 메소드가 비어있는 Reservation 객체를 최초 한번 생성한 이후에 유저 세션에 저장되기 때문에 모든 액션은 동일한 객체를 참조할 수 있다.



### 폼 데이터 검증하기

- 스프링은 Validator 인터페이스를 구현한 validator를 지원한다.

```java
@Component
public class ReservationValidator implements Validator {
    
    @Override
    public boolean supports(Class<?> clazz) {
        return Reservation.class.isAssignableFrom(Clazz);
    }
    
    @Override
    public void validate(Object target, Errors errors) {
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "courtName",
                                                "required.courtName", 
                                                "Court name is required.");
        ValidationUtils.rejectIfEmpty(errors, "date", "required.date", 
                                        "Date is required.");
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "player.name",
                                                "required.playerName", 
                                                "Player name is required.");
        ValidationUtils.rejectIfEmpty(errors, "sportType", "required.sportType", 
                                        "Sport Type is required.");
        
        Reservation reservation = (Reservation) target;
        LocalDate date = reservation.getDate();
        int hour = reservation.getHour();
        if(date != null) {
            if(date.getDayOfWeek() == DayOfWeek.SUNDAY) {
                if(hour < 9 || hour > 22) {
                    errors.reject("invalid.holidayHour", "Invalid holiday hour.");
                }
            } else {
                if (hour < 9 || hour > 21) {
                    errors.reject("invalid.weekdayHour", "Invalid weekday hour.");
                }
            }
        }
        
    }
}

```

- 필수 입력 필드의 기재 여부는 `ValidationUtils` 클래스에 있는 `rejectIfEmptyOrWhitespace()`, `rejectIfEmpty()` 등의 유틸리티 메서드로 조사해 하나라도 값이 비어있으면 에러를 만들어 해당 필드에 바인딩한다.

    - 이들 메서드의 인자는 1번째 - 에러, 2번째  프로퍼티명, 3번째 - 에러 코드, 4번째 - 기본 에러 메세지이다.

- 밑 부분은 유저가 예약 신청한 시간이 운영 시간 이내인지 체크하는 코드로 예약 시간이 올바르지 않으면 `reject()` 로 에러를 만들고 필드가 아닌 Reservation 객체에 바인딩한다.



```java
private ReservationValidator reservationValidator;

@RequestMapping(method=RequestMethod.POST)
public String submitForm(@ModelAttribute("reservation") @Validated Reservation 																						reservation,
                         BindingResult result, SessionStatus sessionStatus) {
    if (result.hasErrors()) {
        return "reservationForm";
    } else {
        reservationService.make(reservation);
        return "redirect:reservationSuccess";
    }
}

@InitBinder
public void initBinder(WebDataBinder binder) {
    binder.setValidator(reservationValidator);
}

```

- Validator를 선언한 뒤, 핸들러 메소드의 @ModelAttribute 뒤에 `@Validated` 를 적어 검증 대상인 것을 확인한다.
    - 검증 결과는 모두 `BindingResult` 에 저장되므로  hasError()로 검사해 분기처리합니다.
- `@InitBinder`를 붙인 메소드는 `setValidator()`로 Binding이후에 사용 될 수 있게 WebDataBinder에 Vaildator를 등록한다. 
    - `addValidators()` 를 사용하면 여러 validator를 한번에 등록할 수 있다.



### 컨트롤러의 세션 데이터 만료시키기

- 폼이 여러 번 전송되거나 유저입력 데이터가 유실되지 않게 하기 위해 컨트롤러에 `@SessionAttributes` 를 붙여서 사용한다.

    - 여러 요청을 거치더라도 Reservation 객체 형태의 예약 필드를 참조가 가능하다.

- 폼 전송 후에도 객체를 저장할 필요는 없기 때문에 `@SessionAttributes` 에 저장된 객체를 `@SessionStatus` 객체로 지운다.

```java
@Controller
@RequestMapping("/reservationForm")
@SessionAttribute("reservation")
public class ReservationFormController {
    @RequestMapping(method=RequestMethod.POST)
    public String submitForm (@ModelAttribute("reservation") Reservation reservation,
                                BindingResult result, SessionStatus status) {
        if(result.hasErros()) {
            return "reservationForm";
        } else {
            reservationService.make(reservation);
            status.setComplete();
            return "redirect:reservationSuccess";
        }
    }
}

```

    

## 레시피 3-10 마법사 폼 컨트롤러로 다중 페이지 폼 처리하기

- 처리해야할 폼이 여러페이지에 걸쳐 있는 경우도 있는데, 이를 __마법사 폼__이라고 한다.

    - 폼 컨트롤러에도 페이지 뷰를 여러 개 정의한다.
    - 컨트롤러는 전체 페이지에 걸쳐 폼 상태를 관리하고 폼을 처리하는 메소드를 하나만 둘 수도 있지만, 액션을 분간하기 위해 전송 버튼과 같은 이름의 특수한 요청 매개변수를 폼마다 두고 관리한다.
        - ___finish__ : 마법사 폼을 마친다.
        - ___cancel__ : 마법사 폼을 취소한다.
        - ___targetx__ : 대상 페이지로 넘어간다 (x는 0부터 시작하는 페이지 인덱스)

    

    

- 다중 폼 처리하는 일 있을 때 보면 좋을듯ㅋ 레퍼런스 수준



## 레시피 3-11 표준 애너테이션(JSR-303)으로 빈 검증하기

- JSR-303 : 자바 빈에 애너테이션을 붙여 검증하는 방법을 표준화한 명세이다.

    - 클래스 패스에 구현체에 대한 선언을 추가해야한다. (Maven/gradle dependency)

    - 도메인 클래스에 애너테이션을 붙여서 검사한다.

```java
public class Reservation {
    @NotNull
    @Size(min=4)
    private String courtName;
    
    @NotNull
    private Date date;
    
    @Min(9)
    @Max(21)
    private int hour;
    
    @Valid
    private Player player;
    
    @NotNull
    private SportType sportType;
    
    // Getter & Setter
}

```

    - 컨트롤러에서 사용하는 방법

```java
@RequestMapping(method=RequestMapping.POST)
public String submitForm(@ModelAttribute("reservation")) @Valid 
                            Reservation reservation, BindingResult result,
                                                        SessionStatus status) {
    if(result.hasErrors()) {
        // 처리                                                      
    }
                                                            
                                                            // 처리
}

```

        - Validator와 코드가 거의 흡사하지만 Validator를 사용하지 않으므로 `@InitBinder` 로 바인딩 해줄 필요없다.
        - SpirngMVC는 Validator가 클래스 패스에 있으면 자동으로 감지한다.
        - `@Valid` 를 붙인 reservation 객체를 검사하고 나머지는 validator랑 똑같이 작동한다.





* 스프링5 레시피 요약 