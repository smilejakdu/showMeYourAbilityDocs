# Exception 처리



<figure><img src=".gitbook/assets/스크린샷 2023-12-07 오후 2.46.51.png" alt=""><figcaption></figcaption></figure>

```java
package com.example.showmeyourability.shared.Exception;

import lombok.Data;
import org.springframework.http.HttpStatus;

@Data
public class HttpExceptionCustom extends RuntimeException {
    private boolean ok;
    private String message;
    private HttpStatus httpStatus;

    public HttpExceptionCustom(boolean ok, String message, HttpStatus httpStatus) {
        super(message);
        this.ok = ok;
        this.message = message;
        this.httpStatus = httpStatus;
    }
}

```

exception 처리를 하게 되면 제일 먼저 여기로 들어오게 된다.

ExceptionResponse 객체는&#x20;

```java
package com.example.showmeyourability.shared.Exception;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class ExceptionResponse {
    private Boolean ok;
    private String message;
    private int statusCode;
}

```

이렇게 작성하고 ErrorHandler 가 필요하다.

```java
package com.example.showmeyourability.shared;

import com.example.showmeyourability.shared.Exception.ExceptionResponse;
import com.example.showmeyourability.shared.Exception.HttpExceptionCustom;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

@ControllerAdvice
public class ErrorHandler {

    @ExceptionHandler(HttpExceptionCustom.class)
    public ResponseEntity<ExceptionResponse> handleException(HttpExceptionCustom e) {
        HttpStatus status = e.getHttpStatus();

        ExceptionResponse exceptionResponse = new ExceptionResponse(
                false,
                e.getMessage(),
                status.value()
        );
        return new ResponseEntity<>(exceptionResponse, status);
    }
}
```

하게 되면 원하는 format 으로 errorHandler 를 출력할 수 있다.

