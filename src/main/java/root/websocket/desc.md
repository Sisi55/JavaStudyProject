# WebSocket 채팅

> https://asfirstalways.tistory.com/359

build.gradle

``` gradle
dependencies{
	compile('org.springframework.boot:spring-boot-starter-web')
	compile('org.springframework.boot:spring-boot-starter-websocket')
	compile('org.webjars.sockjs-client:1.0.2')
	compile('org.webjars.stomp-websocket:2.3.3')
	runtime('org.springframework.boot:spring-boot-devtools')
	testCompile('org.springframework.boot:spring-boot-starter-test')
}
```

WebSocketConfig.java

Message Broker 에 대한 설정을 한다.

`@Override configureMessageBroder()` 의

`enableSimpleBroker()` 는, 메모리 기반 메시지 **브로커가** 해당 api 를 구독하고 있는 **client 에게** 메시지를 전달한다

`setApplicationDestinationPrefixes()` 는, **서버에서 클라이언트로부터**의 메시지를 받을 api prefix 설정



`@Override registerStompEndpoints()` 는 client 에서 WebSocket 을 연결할 api 를 설정한다.

``` java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndoints(StompEndpointRegistry registry){
        registry.addEndpoint("/websockethandler").withSockJS();
    }
}
```

MessageHandler.java

``` java
@RestController
public class MessageHandler{
    
    @MessageMapping("/hello")
    @SendTo("/topic/roomId")
    public Message broadcasting(ClientMessage message) throws Exception{
        return new Message(message.getContent());
    }
}
```

index.html 

``` html
<body>
    <textarea id="chatOutput" name="" class="chatting_history" rows="24"></textarea>
    <div class="chatting_input">
        <input id="chatInput" type="text" class="chat"/>
    </div>
    
    <script src=".../sockjs.min.js"></script>
    <script src="...stomp.min.js"></script>
    <script src="...main.js"></script>
    <script src="...socket.js"></script>
</body>
```

socket.js

``` javascript
document.addEventListener("DOMContentLoaded", function(){
    WebSocket.init();
});

let WebScoket = (function(){
    const SERVER_SOCKET_API = "/websockethandler";
    const ENTER_KEY = 13;
    let stompClient;
    let textArea = document.getElementById("chatOutput");
    let inputElm = document.getElementById("charInput");
    
    function connect(){
        let socket = new SockJs(SERVER_SCOEKT_API); // url
        stompClient = Stomp.over(socket);
        stompClient.connect({}, function(){
            stompClient.subscribe('/topic/roomId', function(msg){
                printMessage(KSON.parse(msg.body).content);
            });
        });
    }
    
    function pringMessage(message){
        textArea.value += message + "\n";
    }
    
    function chatKeyDownHandler(e){
        if(e.which === ENTER_KEY && inputElm.value.trim() !== ""){
            sendMessage(inputElm.value);
            clear(inputElm);
        }
    }
    
    function clear(input){
        input.value = "";
    }
    
    function sendMessage(text){
        stompClient.send("/app/hello",{}, JSON.stringify({'content':text}));
    }
    
    return {
        init:init
    }
});
```



# SB WebSocket

> https://handcoding.tistory.com/171



세션 공유를 위해 redis 와 json 으로 저장하기 위해

gson, websocket 추가



redis config

``` properties
# Redis
spring.redis.host=192.168.85.128
spring.redis.password=handcoding
spring.redis.post=6379
spring.redis.database=1
```



``` java
@Configuration
public class RedisConfig{
    
    @Bean
    public StringRedisTemplate redisTemplate(JedisConnectionFactory jedisConnectionFactory){
        StringRedisTemplate stringRedisTemplate = new StringRedisTemplate();
        stringRedisTemplate.setConnectionFactory(jedisConnectionFactory);
        return stringRedisTemplate;
    }
}
```



redis session 작성

``` java
@Component
public class RedisSession{
    
    @Resource(name="redisTemplate")
    private ValueOperations<String,String> tokens;
    
    private Gson gson = new Gson();
    
    public String getToken(OutLoginVO outLoginVO){
        TempKey tempKey = new TempKey();
        String newToken = tempKey.getKey(300);
        String oldToken = getCheckId(outLoginVO);
        String token = null;
        
        if(oldToken == null){
            token = newToken;
        }else{
            token = oldToken;
        }
        setToken(token, outLoginVO);
        return token;
    }
    
    private String getCheckId(OutLoginVO outLoginVO){
        String oldToken = (String) tokens.get(outLoginVO.getId()+outLoginVO.getDomain());
        return oldToken;
    }
    
    private void setToken(String token, OutLoginVO outLoginVO){
        if(outLoginVO.getTimeout()==0 || outLoginVO.getTimeUnit()=null){
    tokens.set(token,gson.toJson(outLoginVO),30,TimeUnit.MINUTES);
            tokens.set(outLoginVO.getId()+outLoginVO.getDomain(),token,30,TimeUnit.MINUTES);
        }else{
            tokens.set(token.gson.toJson(outLoginVO),outLoginVO.getTimeout(),outLogin.getTimeUnit());
            tokens.set(outLoginVO.getId()+outLoginVO.getDomain(),token,outLoginVO.getTimeout(),outLoginVO.getTimeUnit());
        }
    }
    
    public OutLoginVO getUserVO(String token){
        OutLoginVO outLoginVO = gson.fromJson(tokens.get(token),OutLoginVO.class);
        if(outLoginVO != null){
            setToken(token,outLoginVO);
        }
        return outLoginVO;
    }
}
```



websocket 설정

시작 URL, 크로스 도메인, sendURL, responseURL 설정한다.

``` java
@Configuration
@EnableWebScoketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer{
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config){
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry){
        registry.addEndpoint("/websocket").setAllowedOrigins("*").withSockJS();
    }
}
```



InVO

``` java
public class InChatMessageVO{
    // 사용자 정보 가져올 토큰
    private String token;
    // 메세지 내용
    private String content;
    // 채팅 타입
    private String type;
    // 채팅 고유 아이디
    private String charId;
}
```



컨트롤러

``` java
@Controller
public class ChatController {
    
    @Autowired
    private SimpMessagingTemplate simpMessagingTemplate;
    
    // redis 세션
    @Autowired
    private RedisSession redisSession;
    
    protected static final Logger logger = LoggerFactory.getLogger(CharController.class);
    
    @MessageMapping("/chats")
    public void greeting(InChatMessageVO inMs) throws Exception {
        OutChatMessageVo outMs = new OutChatMessageVO();
        
        // 사용자 세션 정보 가져오기
        OutLoginVO info = redisSession.getUserVO(inMs.getToken());
        logger.info(into.toString());
        
        boolean check = false;
        if(info != null){
            outMs.setId(info.getId());
            outMs.setContent(info.getContent());
            outMs.setDomain(info.getDomain());
            outMs.setName(info.getName());
            
            check = true;
        }
        
        if(check){
            // 메시지 실시간 응답
            this.simpMessagingTemplate.convertAndSend("/topic/chats."+inMs.getType()+inMs.getChatId(), outMs);
        }
    }
}
```



front 로그인 시 처리

``` java
// 셋팅
@GetMapping("/")
public String view(HttpSession session){
    // front 서버 세션 정보
    LoginVO login = new LoginVO();
    login.setId("test_id");
    login.setDomain("user");
    login.setName("sisi");
    
    // 채팅서버와 공유할 세션 정보
    OutLoginVO outLoginVO = new OutLoginVO();
    outLoginVO.setDomain(login.getDomain());
    outLoginVO.setName(login.getName());
    outLoginVO.setId(login.getId());
    
    login.setToken(redisSession.getToken(outLoginVO));
    
    // front 세션 저장
    session.setAttribute("user",login);
    
    return "chat";
}
```



char.js

``` javascript
var websocket = {
    stompClient: null,
    token: null,
    type: null,
    chatId: null
};

function connect(token, type, chatId, fn){
    websocket.token = token;
    websocket.type = type;
    websocket.charId = charId;
    var socket = new SockJS('http://localhost:8080/websocket');
    websocket.stompClient = Stomp.over(socket);
    websocket.stompClient.connect({}, function(frame){
        console.log('Connected: '+frame);
        websocket.stompClient.subscribe('/topic/chats.'+type+charId, function(obj){
            fn(JSON.parse(obj.body));
        });
    });
}

function disconnect(){
    if(stompClient !== null){
        websocket.stompClient.disconnect();
    }
    console.log("Disconnected");
}

function sendContent(content){
    var query = {}
    query.token = websocket.token;
    query.type = websocket.type;
    query.charId = websocket.charId;
    query.content = content;
    websocket.stompClient.send('/app/chars',{}, JSON.stringify(query));
}
```



front session 에 저장된 token 정보 셋팅

``` html
<input type="hidden" id="hiddenToken" value="${user.token}"/>
```



main.js

페이지 로딩 시 바로 웹 소켓 서버와 접속

``` javascript
$(function(){
    
    // 소켓 서버 접속, 정보 셋팅, 응답 fn 정의
    connect($('#hiddenToken').val(), 'order','1', function(obj){
        var message = obj.domain+':'+obj.id+'('+obj.name+')'+' - '+obj.content;
        $('#greetings').append('<tr><td>'+message+'</td></tr>');
        $('#name').val('');
    });
    
    // 메시지 전송
    $('#send').click(function(){
        sendContent($('#name').val());
    })
});
```



sb stomp 가 url 로 소켓 세션 관리를 해주어 편하다



# Spring WebSocket

> https://supawer0728.github.io/2018/03/30/spring-websocket/



WebSocket 기본 설정

``` java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer{
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry){
        registry.addHandler(new MyHandler(), "/myHandler")
            .setAllowedOrigins("*").withSockJS();
    }
}
```



// mustache !!



WebSocketConfig

``` java
@Profile("!stomp")
@configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    
    @Autowired
    private ChatHandler chatHandler;
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry){
        // 해당 endpoint 로 handshake 가 이루어진다.
        registry.addHandler(charHandler, "/ws/char")
            .setAllowedOrigins("*").withSockJS();
    }
}
```



CharRoom

``` java
@Getter
public class ChatRoom {
    private String id;
    private String name;
    private Set<WebSocketSession> sessions = new HashSet<>();
    
    // 채팅방 생성
    public static ChatRoom create(@NonNull String name){
        ChatRoom created = new ChatRoom();
        created.id = UUID.randomUUID().toString();
        created.name = name;
        return created;
    }
}
```



ChatRoomRepository

``` java
@Repository
public class ChatRoomRepository {
    private final Map<String,ChatRoom> chatRoomMap;
    
    public ChatRoomRepository(){
        chatRoomMap = Collections.unmodifiableMap(Stream.of(
            	ChatRoom.create("1번방"), ChatRoom.create("2번방"), 
            	ChatRoom.create("3번방"))
        	.collect(Collectors.toMap(CharRoom::getId, Function.identity())));
        
        chatRooms = Collections.unmodifiableCollection(chatRoomMap.values());
    }
    
    public ChatRoom getChatRoom(String id){
        return chatRoomMap.get(id);
    }
    
    public Collection<ChatRoom> getChatRooms(){
        return chatRoomMap.values();
    }
}
```



ChatController

``` java
@Controller
@RequestMapping("/chat")
public class ChatRoomController {
    private final ChatRoomRepository repository;
    private final AtomicInteger seq = new AtomicInteger(0);
    
    @Autowired
    public ChatRoomController(ChatRoomRepository repository){
        this.repository = repository;
    }
    
    @GetMapping("/rooms")
    public String rooms(Model model){
        model.addAttribute("rooms", repository.getChatRooms());
        return "/chat/room-list";
    }
    
    @GetMapping("/rooms/{id}")
    public String room(@PathVariable String id, Model model){
        ChatRoom room = repository.getChatRoom(id);
        model.addAttribute("room", room);
        model.addAttribute("member","member"+seq.incrementAndGet());
        // 회원 이름 부여
        
        return "/chat/room";
    }
}
```



room-list.html

``` html
{{#rooms}}
	<li><a href="/chat/rooms/{{id}}">{{name}}</a></li>
{{/rooms}}
```



``` javascript
<script>
$(function(){
    var chatBox = $('.chat_box');
    var messageInput = $('input[name="message"]');
    var sendBtn = $('.send');
    var roomId = $('.content').data('room-id');
    var member = $('.content').data('member');
    
    // handshake
    var sock = new SockJS("/ws/chat");
    
    // onopen : connection 이 맺어졌을 때의 callback
    sock.onopen = function(){
        
        // send:connection 으로 message 전달
        // connection 이 맺어진 후 가입 JOIN 메시지 전달
        sock.send(JSON.stringify({
            chatRoomId: roomId,
            type: 'JOIN',
            writer: member
        }));
        
        // onmessage: message 를 받았을 때의 callback
        sock.onmessage = function(e){
            var content = JSON.parse(e.data);
            chatBox.append('<li>'+ content.message +'('+content.writer+')</li>');
        }
    };
    
    sendBtn.click(function(){
        var message = messageInput.val();
        
        sock.send(JSON.stringify({
            chatRoomId: roomId,
            type: 'CHAT',
            message: message,
            writer: member
        }));
        
        messageInput.val('');
    });
});
</script>    
```



ChatMessage

``` java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class ChatMesage{
    private String chatRoomId;
    private String writer;
    private String message;
    private MessageType type;
}
```



CharHandler

``` java
@Slf4j
@Profile("!stomp")
@Component
public class ChatHandler extends TextWebSocketHandler {
    private final ObjectMapper objectMapper;
    private final ChatRoomRepository repository;
    
    @Autowired
    public ChatHandler(ObjectMapper objectMapper, ChatRoomRepository chatRoomRepository){
        this.objectMapper = objectMapper;
        this.repository = chatRoomRepository;
    }
    
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String payload = message.getPayload();
        log.info("payload: {}", payload);
        
        ChatMessage chatMessage = objectMapper.readValue(
            payload, ChatMessage.class);
        
        ChatRoom chatRoom = repository.getChatRoom(chatMessage.getChatRoomId());
        chatRoom.handleMessage(session, chatMessage, objectMapper);
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        repository.remove(session);
    }
}
```

ChatRoom

``` java
@Getter
public class ChatRoom {
    private String id;
    private String name;
    private Set<WebSocketSession> sessions = new HashSet<>();
    
    public void handleMessage(WebSocketSession session, ChatMessage chatMessage, ObjectMapper objectMapper){
        
        if(chatMessage.getType() == MessageType.JOIN){
            join(session);
            chatMessage.setMessage(chatMessage.getWriter()+"님이 입장했습니다.");
        }
        send(chatMessage, objectMapper);
    }
    
    private void join(WebSocketSession session){
        sessions.add(session);
    }
    
    private <T> void send(T messageObject, ObjectMapper){
        TextMessage message = new TextMessage(
            objectMapper.writeValueAsString(messgeObject));
        sessions.parallelStream().forEach(session -> session.sendMessage(message));
    }
}
```



지금까지 WebSocket 의 기본 동작을 살펴봄.

Spring에서 지원하는 STOMP 를 사용하면 단점 개선됨.



StompWebSocketConfig

``` java
@Profile("stomp")
@EnableWebSocketMessageBroker
@Configuration
public class StompWebSocketConfig implements WebScoketMessageBrokerConfigurer {
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry){
        // handshake 와 통신 담당할 endpoint 지정함
        // client 에서 설정함!
        registry.addEndpoint("/stomp-chat")
            .setAllowedOrigins("*").withSockJS();
    }
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry){ // Application 내부에서 사용할 path 지정 가능
        // client 에 SEND 요청 처리
        registry.setApplicationDestinationPrefixes("/publish");
        // 해당 경로 subscribe 하는 client 에세 메시지 전달
        registry.enableSimpleBroker("/subscribe");
    }
}
```



room.html

``` html
<body>
    <div class="content" data-room-id="{{room.id}}" data-member="{{member}}">
        <ul class="chat_box">
            <input name="message">
            <button class="send">보내기</button>
        </ul>
    </div>
    
    <script>
// ...
    </script>
</body>
```



``` javascript
// 위의 ...에 삽입될 내용.
$(function(){
    var chatBox = $('.chat_box');
    var messageInput = $('input[name="message"]');
    var sendBtn = $('.send');
    var roomId = $('.content').data('room-id');
    var member = $('.content').data('member');
    
    var sock = new SockJS("/stomp-chat");
    // 1. SockJS를 내부에 담고 있는 client 를 내어준다
    var client = Stomp.over(sock);
    
    // 2. connection 이 맺어지면 실행된다.
    client.connect({}, function(){
        // 3. send(path,header,message)로 메시지를 보낼 수 있다.
        client.send('/publish/chat/join', {},
                   JSON.stringify({
            chatRoomId: roomId,
            writer: member
        }));
        
        // 4. subscribe(path,callback)으로 메시지를 받을 수 있다
        // callback의 매개1에 메시지의 내용이 들어온다
        client.subscribe('/subscribe/chat/room'+roomId,
        	function(chat){
            	var content = JSON.parse(chat.body);
            	chatBox.append('<li>'+content.message+'('+content.writer+')</li>');
        	}                
        );
    });
    
    sendBtn.click(function(){
        var message = messageInput.val();
        client.send('/publish/chat/message',{},
                   JSON.stringify({
            chatRoomId: roomId,
            message: message,
            writer: member
        }));
    })
});
```



controller

``` java
@Profile("stomp")
@Controller
public class ChatMessageController {
    
    private final SimpleMessagingTemplate template;
    
    @Autowired
    public ChatMessageController(SimpleMessagingTemplate template){
        this.template = template;
    }
    
    @MessageMapping("/chat/join")
    public void join(ChatMessage message){
        message.setMessage(message.getWriter()+"님이 입장하셨습니다.");
        template.convertAndSend(
            "/subscribe/chat/room/"+message.getChatRoomId(), 
            message);
    }
    
    @MessageMapping("/chat/message")
    public void message(ChatMessage message){
        template.convertAndSend(
            "/subscribe/chat/room"+message.getChatRoomId(), 
            message);
    }
}
```

# SB, STOMP WebSocket 사용

> https://hwiveloper.github.io/2019/01/10/spring-boot-stomp-websocket/



HelloMessage.java

``` java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class HelloMessage{
    private String name;
}
```

Greeting.java

``` java
@NoArgsConstructor
@AllArgsConstructor
@Getter
public class Greeting{
    private String content;
}
```

GreetingController.java

``` java
@Controller
public class GreetingController{
    
    @MessageMapping("/hello")
    @SendTo("/topic/greetings")
    public Greeting greeting(HelloMessage message) throws Exception {
        Thread.sleep(100);
        return new Greeting("Hello, "+ HtmlUtils.htmlEscape(message.getName())+"!");
    }
}
```



> [https://start.goodtime.co.kr/2014/06/%EC%95%8C%EC%95%84%EB%91%90%EB%A9%B4-%ED%8E%B8%EB%A6%AC%ED%95%9C-%EC%9E%90%EB%B0%94-%EC%9C%A0%ED%8B%B8%EB%A6%AC%ED%8B%B0-%ED%81%B4%EB%9E%98%EC%8A%A4%EB%93%A4/](https://start.goodtime.co.kr/2014/06/알아두면-편리한-자바-유틸리티-클래스들/)
>
> 1. `HtmlUtils` 클래스.
>
> 대표적인 메서드가 `String htmlEscape(String input)`. 
>
> `<` 기호를 `<`로, `&` 기호를 `&`로 바꾸는 등의 작업을 처리해준다. 
>
> 사용자가 입력한 문자열에 HTML 코드가 섞여 있으면 문제가 될 경우 이러한 이스케이프 처리는 필수.



``` java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer{
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry){
        // client 로 메시지 응답할 때의 prefix 정의
        registry.enableSimpleBroker("/topic");
        // client 에서 메시지 받을 때의 prefix 정의
        registry.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry){
        // client 에서 SockJS 생성자에 설정하여 연결
        registry.addEnpoint("/websocket").withSockJS();
    }
}
```



index.html



app.js

``` javascript
var stompClient = null;

function setConnected(connected){
    $('#connect').prop("disabled", connected);
    $('#disconnect').prop("disabled", !connected);
    
    if(connected){
        $("#conversation").show();
    }else{
        $("#conversation").hide();
    }
    
    $("#greetings").html("");
}

function connect(){
    var socket = new SockJS('/websocket');
    stompClient = Stomp.over(socket);
    
    // SockJS 와 stomp client 를 통해 연결 시도
    stompClient.connect({}, function(frame){
        setConnected(true);
        console.log('Connected: '+frame);
        
        stompClient.subscribe('/topic/greetings', function(greeting){
            showGreeting(JSON.parse(greeting.body).content);
        });
    });
}

function disconnect(){
    if(stompClient !== null){
        stompClient.disconnect();
    }
    setConnected(false);
    console.log("Disconnected");
}

function sendName(){
    stompClient.send("/app/hello", {},
                    JSON.stringify({
        'name': $("#name").val()
    }));
}

function showGreeting(message){
    $("#greetings").append("<tr><td>"+message+"</td></tr>");
}

$(function(){
    $("form").on('submit', function(e){
        e.preventDefault();
    });
    
    $("#connect").click(function(){ connect(); });
    $("#disconnect").click(function(){ disconnect(); });
    $("#send").click(function(){ sendName(): });
});
```



간단한 채팅 어플리케이션 만들어보기

Chat.java

``` java
@NoArgsConstructor
@AllArgsConstructor
@Getter
@Setter
public class Chat{
    private String name;
    private String message;
}
```

GreetingController.java

``` java
@MessageMapping("/chat")
@SendTo("/topic/chat") // client 가 subscribe 함
public Chat chat(Chat chat) throws Exception{
    return new Chat(chat.getName(), chat.getMessage());
}
```



> github
>
> https://github.com/hwiVeloper/SpringBootStudy/tree/master/spring-boot-websocket

# WebScoket

> https://hyeooona825.tistory.com/89

``` gradle
compile('org.springframework.boot:spring-boot-starter-websocket')
compile('org.webjars:stomp-websocket:2.3.3')
```



``` java
@Configuration
@EnableWebScoketMessageBroker
public class WebScoketConfig implements WebScoketMessageBrokerConfigurer{
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry){
        // /topic 은 1:N
        // /queue 는 1:1 !!
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry){
        registry.addEndpoint("room1").addInterceptors(
        new HttpHandlshakeInterceptor()).withSockJS();
    }
}
```

> addInterceptor ? // 일단 그냥 인터셉터 추가
>
> HttpHandlshakeInterceptor ? // 아래 정의



``` java
public class HttpHandshakeInterceptor implements HandShakeInterceptor{
    
    @Override
    public boolean beforeHandshake(
    	ServerHttpRequest request,
        ServerHttpResponse response,
        WebSocketHandler wsHandler,
        Map sttributes
    ){
        if(request instanceof ServletServerHttpRequest){
            ServletServerHttpRequest servletRequest = (ServletServerHttpRequest) request;
            HttpSession session = servletRequest.getServeltRequset().getSession();
            attributes.put(SESSION, session);
        }
        
        return true;
    }
    
    @Override
    public void afterHandshake(
    	ServerHttpRequest request,
        ServerHttpResponse response,
        WebSocketHandler wsHandler,
        Exception ex
    )
}
```



``` java
@Controller
public class ChatController{
    
    @MessageMapping("info")
    @SendToUser("/queue/info")
    public String info(
    	String message,
        SimpleMessageHeaderAccessor messageHeaderAccessor
    ){
        User talker = messageHeaderAccessor
            .getSessionAttributes()
            .get(SESSION)
            .get(USER_SESSION_KEY);
        
        return message;
    }
    
    @MessageMapping("chat")
    @SendTo("/topic/message")
    public String chat(
    	Stirng message,
        SimpleMessageHeaderAccessor messageHeaderAccessor
    ){
        User talker = messageHeaderAccessor
            .getSessionAttributes()
            .get(SESSION)
            .get(USER_SESSION_KEY);
        
        if(talker == null)
            throw new UnAuthenticationException("로그인한 사용자만 채팅에 참여할 수 있습니다!");
        
        return message;
    }
    
    @MessageMapping("bye")
    @SendTo("/topic/bye")
    public User bye(String message){
        User talker = messageHeaderAccessor
            .getSessionAttributes()
            .get(SESSION)
            .get(USER_SESSION_KEY);
        
        return talker;
    }
}
```

# SB, WebSocket

> https://lahuman.jabsiri.co.kr/202

WebSocket Configuration

``` java
@Configuration
@EnableWebSocketMessageBroker
public class WebScoketConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry){
        // client 에서 ws 에 접속하는 endpoint 등록
        registry.addEndpoint("/ws").withSockJS();
    }
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry){
        // 해당 경로로 들어오는 메시지만 핸들러로 라우팅
        registry.setApplicationDestinationPrefixes("/app");
        // 해당 경로로 방송되는 메시지만 라우팅 (-> client)
        registry.enableSimpleBroker("/topic");
    }
}
```



ChatController

``` java
@Controller
public class ChatController{
    
    @MessageMapping("/chat/sendMessage")
    @SendTo("/topic/public")
    public ChatMessage sendMessage(
    	@Payload ChatMessage chatMessage
    ){
        return chatMessage;
    }
    
    @MessageMapping("/chat/addUser")
    @SendTo("/topic/public")
    public ChatMessage addUser(
    	@Payload ChatMessage chatMessage,
        SimpleMessageHeaderAccessor headerAccessor
    ){
        // ws session 에 user 추가
        headerAccessor.getSessionAttributes()
            .put("username", chatMessage.getSender());
        return chatMessage;
    }
}
```



main.js

``` javascript
function connect(event){
    username = document.querySelector('#name').value.trim();
    
    if(username){
        usernamePage.classList.add("hidden");
        chatPage.classList.remove("hidden");
        
        var socket = new SockJS("/ws");
        stompClient = Stomp.over(socket);
        
        stompClient.connect({}, onConnected, onError);
    }
    
    event.preventDefault();
}

function onConnected(){
    stompClient.subscribe(
        '/topic/public', onMessageReceived);
    
    stompClient.send("/app/chat/addUser",{},
                    JSON.stringify({
        sender: username,
        type: 'JOIN'
    }));
    
    connectingElement.classList.add('hidden');
}
```

# SB, WebSocket

> https://heowc.tistory.com/10

gradle

``` gradle
compile('org.springframework.boot:spring-boot-starter-websocket')
compile('org.webjars:stomp-websocket:2.3.3')
```

model

``` java
@Getter
@Setter
public class HelloMessage{
    private String name;
    private String contents;
    private Date sendDate;
}
```

controller

``` java
@Controller
public class ChatController{
    
    @MessageMapping("hello")
    @SendTo("/chat/hello")
    public HelloMessage hello(HelloMessage message) throws Exception{
        Thread.sleep(100);
        return message;
    }
    
    @MessageMapping("bye")
    @SendTo("/chat/bye")
    public HelloMessage bye(HelloMessage message) throws Exception {
        Thread.sleep(100);
        return message;
    }
    
    @MessageMapping("detail")
    @SendTo("/chat/detail")
    public HelloMessage detail(HelloMessage message) throws Exception{
        Thread.sleep(100);
        message.setSendDate(new Date());
        return message;
    }
}
```

config

``` java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer{
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config){
        config.enableSimpleBroker("/chat");
        config.setApplicationDestinationPrefixes("/");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry){
        registry.addEndpoint("/wonchul-websocket").withSockJS();
    }
}
```

script

``` javascript
// 소켓 연결
function connect(){
    var socket = new SockJS('/wonchul-websocket')
    stompClient = Stomp.over(socket);
    
    stompClient.connect({}, function(frame){
        setConnected(true);
        console.log('Connected: '+frame);
        
        // 입장에 대한 구독
        stompClient.subscribe('/chat/hello', function(msg){
            showHello(JSON.parse(msg.body));
        });
        
        // 입장에 대한 메시지 전달
        stompClient.subscribe('/chat/detail', function(msg){
            showDetail(JSON.parse(msg.body));
        });
        
        // 퇴장에 대한 구독
        stompClient.subscribe('/chat/bye', function(msg){
            showBye(JSON.parse(msg.body));
        });
        
        sendHello();
    });
}

// 소켓 연결 끊음
function disconnect(){
    if(stompClient != null){
        sendBye();
        stompClient.disconnect();
    }
    
    setConnected(false);
    console.log("Disconnected");
}

// 메시지 전달
function sendHello(){
    stompClient.send("/hello", {}, JSON.stringify({
        name: $("#name").val()
    }));
}

function sendDetail(){
    stompClient.send("/detail", {}, JSON.stringify({
        name: $("#name").val(),
        contents: $("#btn-input").val()
    }));
}

function sendBye(){
    stompClient.send("/bye", {}, JSON.stringify({
        name: $("#name").val()
    }));
}

// 구독한 데이터 표현
function showHello(message){
    var html = "..." + message;
    
    $(".chat").append(html);
    $(".panel-body").scrollTop($(".chat")[0].scrollHeight);
}

```

> github
>
> https://github.com/heowc/SpringBootSample/tree/master/SpringBootWebSocket

> https://m.blog.naver.com/PostView.nhn?blogId=scw0531&logNo=221097188275&proxyReferer=https%3A%2F%2Fwww.google.com%2F

``` javascript
document.addEventListener("DOMContentLoaded", function(){
    WebSocket.init(); // 꼭 이 함수가 아니더라도 이벤트 활용해보기!
})
```

> DOM 이벤트 자료
>
> 1. https://developer.mozilla.org/ko/docs/Web/API/EventTarget/addEventListener
> 2. https://developer.mozilla.org/ko/docs/Web/Events
>
> + https://javacpro.tistory.com/37
> + https://m.blog.naver.com/PostView.nhn?blogId=qbxlvnf11&logNo=220877806711&proxyReferer=https%3A%2F%2Fwww.google.com%2F

# Spring WebSocket Client

> SB가 client !
>
> https://www.leafcats.com/280

neovisionaries 라이브러리 사용

> https://github.com/TakahikoKawasaki/nv-websocket-client

``` gradle
compile('com.neovisionaries:nv-websocket-client:2.4')
```



``` java
public class EchoClient{
    
    private static final String SERVER = "ws://echo.websocket.org";
    private static final int TIMEOUT = 5000;
    
    public static void main(String[] args) throws Exception{
        // connect to server
        WebScoket ws = connect();
        
        // standard input via BufferedReader
        BufferedReader in = getInput();
        
        String text;
        
        // read lines until exit entered
        while((text = in.readLine()) != null){
            if(text.equals("exit")){
                break;
            }
            ws.sendText(text);
        }
        
        // close ws
        ws.disconnect();
    }
    
    private static WebSocket connect() throws IOException, WebSocketException {
        System.out.println("start get data from upbit");
        
        return new WebSocketFactory()
            .setConnectionTimeout(TIMEOUT)
            .createSocket(SERVER)
            .addListener(new WebSocketAdapter(){
                // binary message arrived from the server
                public void onBinaryMessage(WebSocket websocket, byte[] binary){
                    String str = new String(binary);
                    System.out.println(message);
                }
                
                // text message arrived from the server
                public void onTextMessage(WebSocket websocket, String message){
                    System.out.println(message);
                }
            }).addExtension(
            	WebSocketExtension.PERMESSAGE_DEFALTE)
            .connect();
    }
    
    private static BufferedReader getInput() throws IOException{
        return new BufferedReader(new InputStreamReader(System.in));
    }
}
```



`onTextMessage(), onBinaryMessage()` 메소드를 모두 열어 확인해보자!

# SB, STOMP WebSocket

> async 랜덤 채팅
>
> https://zaccoding.tistory.com/16

### Server

Async 설정

``` java
@Configuration
@EnableAsync
public class AsyncConfiguration{
    
    @Bean
    public Executor asyncThreadPool(){
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(3);
        taskExecutor.setMaxPoolSize(30);
        taskExecutor.setQueueCapacity(10);
        taskExecutor.setThreadNamePrefix("Async-Executor-");
        taskExecutor.setDaemon(true);
        taskExecutor.initialize();
        
        return taskExecutor;
    }
}
```

Stomp 설정

``` java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfiguration implements WebSocketMessageBrokerConfigurer{
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config){
        config.enableSimpleBroker("/topic/chat");
        config.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry){
        registry.addEndpoint("/chat-websocket").withSockJS();
    }
}
```

Join 비동기 처리

``` java
@GetMapping("/join")
@ResponseBody
public DeferredResult joinRequest(){
    String sessionId = ServletUtil.getSession().getId();
    logger.info(">> Join request. session id: {}", sessionId);
    
    final ChatRequest user = new ChatRequest(sessionId);
    final DeferredResult deferredResult = new DeferredResult<>(null);
    chatService.joinChatRoom(user, deferredResult);
    
    deferredResult.onCompletion(() -> 
                                chatService
                                .cancelChatRoom(user));
    deferredResult.onError((throwable) -> 
                           chatService.canceChatRoom(user));
    deferredResult.onTimeout(() -> 
                             chatService.cancelChatRoom(user));
    
    return deferredResult;
}
```

``` java
@Async("asyncThreadPool")
public void joinChatRoom(
	ChatRequest request, DeferredResult deferredResult
){
    logger.info("## Join chat room request. {}[{}]",
               Thread.currentThread().getName(),
               Thread.currentThread().getId);
    
    if(request == null || deferredResult == null)
        return;
    
    try{
        lock.writeLock().lock();
        waitingUsers.put(request, deferredResult);
    } finally{
        lock.writeLock().unlock();
        establishChatRoom();
    }
}
```

웹소켓 연결 이벤트 처리

``` java
@Component
public class WebSocketEventListener {
    private static final Logger logger = LoggerFactory.getLogger(WebSocketEventListener.class);
    
    @Autowired
    private ChatService chatService;
    
    @EventListener
    public void handleWebSocketConnectListener(
    	SessionConnectedEvent event
    ){
        MessageHeaderAccessor accessor = 
            NativeMessageHeaderAccessor.getAccessor(
        		event.getMessage(),
            	SimpleMessageHeaderAccessor.class
        );
        
        GenericMessage generic = (GenericMessage)
            accessor.getHeader("simpleConnectMessage");
        
        Map nativeHeaders = (Map) generic.getHeaders()
            .get("nativeHeaders");
        
        String chatRoomId = ((List) 
            nativeHeaders.get("chatRoomId"))
            .get(0);
        String sessionId = (String) generic.getHeaders()
            .get("simpleSessionId");
        
        chatService.connectUser(chatRoomId, sessionId);
    }
    
    @EventListener
    public void handleWebSocketDisconnectListener(
    	SessionDisconnectEvent event
    ){
        StompHeaderAccessor headerAccessor = 
            StompHeaderAccessor.wrap(event.getMessage());
        String sessionId = headerAccessor.getSessionId();
        chatService.disconnectUser(sessionId);
    }
}
```



웹소켓 @MessageMapping & @SendTo

``` java
@MessageMapping("/chat/message/{chatRoomId}")
public void sendMessage(
    @DestinationVariable("chatRoomId") String chatRoomId,
    @Payload ChatMessage chatMessage
){
    if(!StringUtils.hasText(chatRoomId) || chatMessage == null)
        return;
    
    if(chatMessage.getMessageType() == MessageType.CHAT)
        chatService.sendMessage(chatRoomId, chatMessage);
}
```

ChatService (@SendTo)

``` java
@Service
public class ChatService {
    private static final Logger logger = 
        LoggerFactory.getLogger(ChatService.class);
    private Map waitingUsers;
    private Map connectedUsers;
    private ReentrantReadWriteLock lock;
    
    @Autowired
    private SimpleMessagingTemplate messagingTemplate;
    
    // ...
    public void sendMessage(
    	String ChatRoomId, ChatMessage chatMessage
    ){
        String destination = getDestination(chatRoomId);
        messagingTemplate.convertAndSend(destination, 
                                         chatMessage);
    }
    
    private String getDestination(String chatRoomId){
        return "/topic/chat/"+ chatRoomId;
    }
}
```

### JS

ws connect

``` javascript
ChatManager.connectAndSubscribe = function(){
    
    if(CharManager.stompClient == null || 
       !ChatManager.stompClient.connected){ // 연결 준비x
        
        var socket = new SockJS('/chat-websocket');
        ChatManager.stompClient = Stomp.over(socket);
        
        ChatManager.stompClient.connect({
            chatRoomId: ChatManager.chatRoomId
        }, function(frame){
            console.log('Connected: '+frame);
            ChatManager.subscribeMessage();
        });
    } else{ // 연결 준비o
        ChatManager.subscribeMessage();
    }
}
```

ws send message

``` javascript
ChatManager.sendMessage = function(){
    var $chatTarget = $("#chat-message-input");
    var message = $chatTarget.val();
    $chatTarget.val('');
    
    var payload = {
        messageType: 'CHAT',
        senderSessionId: ChatManager.sessionId,
        message: message
    };
    
    ChatManager.stompClient.send(
    	'/app/chat/message/'+ ChatManager.chatRoomId, {},
        JSON.stringify(payload)
    );
};
```

ws subscribe

``` javascript
ChatManager.subscribeMessage = function(){
    ChatManager.stompClient.subscribe(
    	'/topic/chat/'+ ChatManager.chatRoomId,
        function(resultObj){
            var result = JSON.parse(resultObj.body);
            var message = '';
            
            if(result.messageType == 'CHAT'){
                if(result.senderSessionId == 
                   ChatManager.sessionId){
                    message += '[Me]: ';
                }else{
                    message += '[Anonymous]: ';
                }
                
                message += result.message + '\n';
            }else if(result.messageType == 'DISCONNECTED'){
                message = '>> Disconnected user';
                ChatManager.disconnect();
            }
            
            ChatManager.updateText(message, true);
        }
    );
};
```

> github
>
> https://github.com/zacscoding/spring-websocket-random-chat

