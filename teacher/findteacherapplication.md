# FindTeacherApplication

```java
package com.example.showmeyourability.teacher.application;

import com.example.showmeyourability.comments.domain.QComments;
import com.example.showmeyourability.teacher.domain.QTeacher;
import com.example.showmeyourability.teacher.domain.Teacher;
import com.example.showmeyourability.teacher.infrastructure.dto.FindTeacherDto.FindTeacherByIdResponseDto;
import com.example.showmeyourability.teacher.infrastructure.dto.FindTeacherDto.FindTeacherResponseDto;
import com.example.showmeyourability.teacher.infrastructure.dto.FindTeacherDto.TeacherDto;
import com.example.showmeyourability.teacher.infrastructure.repository.TeacherRepository;
import com.querydsl.core.types.Projections;
import com.querydsl.jpa.impl.JPAQueryFactory;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class FindTeacherApplication {
    private final JPAQueryFactory queryFactory; // JPAQueryFactory 주입
    private final QTeacher qTeacher = QTeacher.teacher; // 클래스 수준의 QTeacher 인스턴스
    private final QComments qComments = QComments.comments; // 클래스 수준의 QComments 인스턴스


    @Transactional
    public FindTeacherResponseDto findAllTeacher(int page, int size) {
        // 교사 정보와 평균 점수를 함께 조회
        List<TeacherDto> teacherDtos = queryFactory
                .select(Projections.constructor(TeacherDto.class,
                        qTeacher.id,
                        qTeacher.career,
                        qTeacher.user.email, // Email 필드 추가
                        qTeacher.skill,
                        qTeacher.user.id,
                        qComments.likes.avg().coalesce(0.0) // 평균 점수 계산 및 null 처리
                ))
                .from(qTeacher)
                .leftJoin(qTeacher.comments, qComments)
                .groupBy(qTeacher.id)
                .orderBy(qTeacher.user.email.asc())
                .offset((long) page * size)
                .limit(size)
                .fetch();

        // 총 페이지 수 계산
        long total = queryFactory
                .selectFrom(qTeacher)
                .fetchCount();

        int lastPage = (int) Math.ceil((double) total / size);

        return convertToFindTeacherResponseDto(lastPage, teacherDtos);
    }

    private FindTeacherResponseDto convertToFindTeacherResponseDto(
            int lastPage,
            List<TeacherDto> teachers
    ) {
        return FindTeacherResponseDto.builder()
                .lastPage(lastPage)
                .teachers(teachers)
                .build();
    }

    @Transactional
    public FindTeacherByIdResponseDto findOneTeacherById(
            Long teacherId
    ) {
        Teacher teacher = queryFactory
                .selectFrom(QTeacher.teacher)
                // Teacher의 ID가 메서드 파라미터로 전달된 teacherId와 같은 경우를 조건으로 합니다.
                .where(QTeacher.teacher.id.eq(teacherId))
                .fetchOne();

//        Teacher teacher = null;
//        if (teacher == null) {
//            throw new NotFoundException("해당 선생님을 찾을 수 없습니다.");
//        }
//        테스트 코드 확인위해서 잠시 주석처리

        FindTeacherByIdResponseDto findTeacherByIdResponseDto = new FindTeacherByIdResponseDto();
        findTeacherByIdResponseDto.setTeacher(teacher);
        return new FindTeacherByIdResponseDto();
    }
}
```

위와 같이 FindTeacherApplication 코드를 작성했다.

위에 대한 테스트 코드는 어떻게 작성하면 좋을까 ?

우선 실패코드를 먼저 작성한 후에 success test 코드를 작성해야 한다 .

```java
package com.example.showmeyourability.service.teacherApplication;

import com.example.showmeyourability.teacher.application.FindTeacherApplication;
import com.example.showmeyourability.teacher.domain.Teacher;
import com.example.showmeyourability.teacher.infrastructure.dto.FindTeacherDto.FindTeacherByIdResponseDto;
import com.example.showmeyourability.teacher.infrastructure.repository.TeacherRepository;
import com.example.showmeyourability.users.domain.GenderType;
import com.example.showmeyourability.users.domain.User;
import com.example.showmeyourability.users.infrastructure.repository.UserRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mock;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.security.crypto.bcrypt.BCrypt;
import org.webjars.NotFoundException;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.Mockito.when;


@ExtendWith(MockitoExtension.class)
public class FindTeacherApplicationTest {

    @Mock // 테스트 대상이 의존하는 객체
    TeacherRepository teacherRepository;
    @Mock // 테스트 대상이 의존하는 객체
    UserRepository userRepository;
    @InjectMock // 테스트 대상
    FindTeacherApplication findTeacherApplication;

    @BeforeEach
    void setUp() {
        String hashedPassword = BCrypt.hashpw("1234", BCrypt.gensalt());

        // User 객체 생성
        User user1 = User.builder()
                .id(1L)
                .email("robertvsd1@gmail.com")
                .genderType(GenderType.MALE)
                .age(20)
                .img("img")
                .password(hashedPassword)
                .build();

        User user2 = User.builder()
                .id(2L)
                .email("robertvsd2@gmail.com")
                .genderType(GenderType.MALE)
                .age(20)
                .img("img")
                .password(hashedPassword)
                .build();

        // userRepository.findById에 대한 모의 동작 정의
        when(userRepository.findById(1L)).thenReturn(Optional.of(user1));
        when(userRepository.findById(2L)).thenReturn(Optional.of(user2));

        // Teacher 객체 생성 및 저장
        Teacher teacher = Teacher.builder()
                .id(1L)
                .user(user1)
                .career("경력")
                .skill("스킬")
                .comments(null)
                .orders(null)
                .build();

         teacherRepository.save(teacher);
    }

    @Test
    void notFoundTeacherByIdTest() {
        // given
        Long teacherId = 3L;

        when(findTeacherApplication.findOneTeacherById(teacherId))
                .thenThrow(new NotFoundException("해당 선생님을 찾을 수 없습니다."));

        // then
        assertThrows(NotFoundException.class, () -> {
            findTeacherApplication.findOneTeacherById(teacherId);
        });        // NotFoundException: 해당 선생님을 찾을 수 없습니다.
    }

    @Test
    void foundTeacherByIdTest() {
        Long teacherId = 1L;

        // 필요한 User 및 Teacher 객체 생성
        User mockUser = new User();
        mockUser.setId(teacherId);
        mockUser.setEmail("robertvsd1@gmail.com");
        // mockUser의 다른 필드도 필요에 따라 설정

        Teacher mockTeacher = new Teacher();
        mockTeacher.setId(teacherId);
        mockTeacher.setUser(mockUser);
        // mockTeacher의 다른 필드도 필요에 따라 설정

        // FindTeacherByIdResponseDto 객체 생성 및 Teacher 객체 설정
        FindTeacherByIdResponseDto mockResponse = new FindTeacherByIdResponseDto();
        mockResponse.setTeacher(mockTeacher);

        // 모의 동작 설정
        when(findTeacherApplication.findOneTeacherById(teacherId)).thenReturn(mockResponse);

        // 테스트 실행
        FindTeacherByIdResponseDto actualResponse = findTeacherApplication.findOneTeacherById(teacherId);

        // 검증
        assertEquals(teacherId, actualResponse.getTeacher().getId());
        assertEquals("robertvsd1@gmail.com", actualResponse.getTeacher().getUser().getEmail());
    }
}

```

위의 테스트 코드에서 @MockBean 을 사용하지 않고 @InjectMock 을 사용했다.

```java

    @Mock // 테스트 대상이 의존하는 객체
    TeacherRepository teacherRepository;
    @Mock // 테스트 대상이 의존하는 객체
    UserRepository userRepository;
    @InjectMock // 테스트 대상
    FindTeacherApplication findTeacherApplication;
```

만약에 @InjectMock 으로 하지않고  @MockBean 으로 했었다면 NullPointerException 에러가 발생했을것이다. 초기화가 되지 않기 때문이다.

@MockBean 어노테이션은 Spring Bean 테스트에서 Spring 컨텍스트 내의 빈을 모의 객체로 교체할 때 사용된다. 그러나 여기서는 @MockBean 대신 @InjectMocks 어노테이션을 사용해야한다.

@InjectMocks 는 Mockito 가 자동으로 필요한 의존성을 주입하여 객체를 초기화하는 데 사용된다.

둘은 차이점이 있는데 헤깔리지말고 적절한 상황에서 맞는어노테이션을 사용해야한다.



`@MockBean`과 `@InjectMocks`는 모두 테스트 환경에서 객체를 모의(Mock) 객체로 만들기 위해 사용되지만, 사용 목적과 적용 방식에 차이가 있습니다.

#### `@MockBean`

* `@MockBean`은 Spring Boot의 테스트 환경에서 사용됩니다.
* 이 어노테이션은 Spring의 애플리케이션 컨텍스트 내에 있는 빈(Bean)을 모의 객체로 교체합니다.
* 주로 Spring Boot의 `@SpringBootTest`와 같이 전체 Spring 컨텍스트를 로드하는 테스트에서 사용됩니다.
* `@MockBean`을 사용하면, 해당 빈은 모든 테스트 케이스와 Spring 컨텍스트 전반에서 모의 객체로 대체됩니다.

#### `@InjectMocks`

* `@InjectMocks`는 Mockito 테스팅 프레임워크에서 사용됩니다.
* 이 어노테이션은 지정된 클래스의 인스턴스를 생성하고, `@Mock` (또는 `@Spy`)으로 주석이 달린 필드를 해당 인스턴스에 자동으로 주입합니다.
* `@InjectMocks`는 주로 단위 테스트에서 사용되며, Spring 컨텍스트를 로드하지 않습니다.
* 이는 테스트 대상 클래스의 실제 인스턴스를 생성하고, 필요한 의존성만 모의 객체로 주입하는 데 사용됩니다.

#### 사용 이유

귀하의 경우에는 `SignUpUserApplicationTest` 클래스에서 `SignupUserApplication`의 인스턴스를 테스트하고 있습니다. 이 클래스는 Spring 컨텍스트에 의존하지 않는 순수한 단위 테스트로 보입니다. 따라서 `@InjectMocks`를 사용하여 `SignupUserApplication`의 실제 인스턴스를 생성하고, 필요한 의존성(`UserRepository`)을 `@Mock`으로 주입하는 것이 적절합니다. 이렇게 하면 테스트 실행 시 `NullPointerException`을 방지할 수 있으며, 테스트의 실행 속도도 빨라집니다.

