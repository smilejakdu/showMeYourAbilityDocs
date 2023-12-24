# FindRecentTeacher

FindRecentTeacherTest 를 작성할때,

`@BeforeEach` 에서 미리 user 와 teacher list 를 생성해줘야한다.



## Mock 불러오기

QueryDSL 를 사용하고 있기때문에 QTeacher 클래스를 가져와서 사용해야한다.

```java
@ExtendWith(MockitoExtension.class)
public class FindRecentTeacherApplicationTest {

    @Mock
    private JPAQueryFactory queryFactory;

    @InjectMocks
    private FindRecentTeacherApplication findRecentTeacherApplication;

    private final QTeacher qTeacher = QTeacher.teacher;


```



`QTeacher` 클래스는 주입된게 아니기 때문에 위와 같이 넣어주도록 하고&#x20;

Mock 은 주입된 객체를 가져오고&#x20;

주입된 객체를 사용하는 클래스에서는&#x20;

InjectMocks 로 작성해주면 된다.



## BeaforeEach 작성

사용을 하려면 미리 user 와 teacher 를 작성해야한다.

```java
@ExtendWith(MockitoExtension.class)
public class FindRecentTeacherApplicationTest {

    @Mock
    private JPAQueryFactory queryFactory;

    @InjectMocks
    private FindRecentTeacherApplication findRecentTeacherApplication;

    private final QTeacher qTeacher = QTeacher.teacher;

    private List<TeacherDto> teacherDtoList;
    private User user;

    LocalDateTime fourDaysAgo = LocalDateTime.now().minusDays(4);
    LocalDateTime threeDaysAgo = LocalDateTime.now().minusDays(3);

    @BeforeEach
    public void setup() {
        String password = "password1234";
        String hashPassword = BCrypt.hashpw(password, BCrypt.gensalt());
        user = User.builder()
                .id(1L)
                .name("robert")
                .email("robert@gmail.com")
                .password(hashPassword)
                .build();


        teacherDtoList = Arrays.asList(
                TeacherDto.builder()
                        .id(1L)
                        .career("career")
                        .skill("skill")
                        .createdAt(fourDaysAgo)
                        .userId(user.getId())
                        .email(user.getEmail())
                        .build(),
                TeacherDto.builder()
                        .id(2L)
                        .career("career2")
                        .skill("skill2")
                        .userId(user.getId())
                        .createdAt(threeDaysAgo)
                        .email(user.getEmail())
                        .build()
        );
    }

```



위와 같이 미리 user 와 teacher 를 작성해준다.



## 성공테스트

원래는 성공테스트 전에 실패테스트를 충분히 해야한다.

하지만 최신 4일동안 가입한 선생님을 찾는거라서 ,&#x20;

실패케이스를 찾지 못해서 우선 성공케이스만 작성하기로 한다. ㅠㅠ



```java
    @Test
    @DisplayName("최근 4일간 가입한 선생님들 조회")
    public void testExecute() {
        // given
        JPAQuery<Teacher> mockQuery = mock(JPAQuery.class);
        when(queryFactory.selectFrom(qTeacher)).thenReturn(mockQuery);
        when(mockQuery.where(any(BooleanExpression.class))).thenReturn(mockQuery);
        when(mockQuery.innerJoin(qTeacher.user)).thenReturn(mockQuery);
        when(mockQuery.fetch()).thenReturn(
                Arrays.asList(
                        Teacher.builder()
                                .id(1L)
                                .career("career")
                                .skill("skill")
                                .createdAt(fourDaysAgo) // createdAt 값을 수동으로 설정
                                .user(user)
                                .build(),
                        Teacher.builder()
                                .id(2L)
                                .career("career2")
                                .skill("skill2")
                                .createdAt(threeDaysAgo) // createdAt 값을 수동으로 설정
                                .user(user)
                                .build()
                )
        );

        // when
        FindRecentTeacherResponseDto findRecentTeacherResponseDto = findRecentTeacherApplication.execute();
        System.out.println("findRecentTeacherResponseDto = " + findRecentTeacherResponseDto);
        // then
        assertEquals(teacherDtoList, findRecentTeacherResponseDto.getTeacherDtoList());
    }

```

처음에는 BaseTimeEntity 를 상속 받아서 사용했는데

상속을 받으니까 Test 코드를 작성하게 될때 @Builder 부분에서 에러가 발생했다.

그래서  Builder 부분을 custom 하게 지금 사용해서 수정을 했습니다.

```java
@Entity
@Getter
@AllArgsConstructor
@Builder
@NoArgsConstructor
@Table(name = "teachers", indexes = {
        @Index(name = "teacher_index", columnList = "id")
})
public class Teacher extends BaseTimeEntitiy {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100, name = "career")
    private String career;

    @Column(nullable = false, length = 100, name = "skill")
    private String skill;

    @OneToOne()
    @JsonIgnore
    @JoinColumn(name = "user_id")
    private User user;

    @JsonIgnore
    @OneToMany(fetch = FetchType.EAGER, mappedBy = "teacher")
    private List<Comments> comments = Collections.emptyList();

    @JsonIgnore
    @OneToMany(fetch = FetchType.EAGER, mappedBy = "teacher")
    private List<Order> orders = Collections.emptyList();

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    private LocalDateTime deletedAt;
}

```

