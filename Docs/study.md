## 1. authModlue 회원가입

---

authModule에서 무엇을 import 하는지 확인.
authContrller 확인

```
  @Post('signup')
  async signup(@Body() createUserDto: CreateUserDto): Promise<SignupResDto> {
    const user = await this.userService.createUser(createUserDto);
    return {
      id: user.id,
      name: user.name,
      email: user.email,
      phone: user.phone,
    };
  }
```

1. @Post 데코레이터를 달아 HTTP요청을 처리 하고 엔드포인트를 /signup 으로 정의

2. 비동기함수 async 로 선언되어 signup 이라는 이름을 주어 @Body 데코레이터로 데이터를 추출

3. createUserDto에 할당 하며 CreateUserDto 타입이다. 

4. 반환타입은 SignupResDto 형식의 Promise를 반환하는것을의미

5. CreateUser메서드를 호출하여 new 생성자를 생성하고 반환받은 사용자 객체를 user 에 할당

6. createUser메서드는 비동기적으로 실행되며, await 키워드를 사용해 해당 작업이 완료될 때가지 기다린다.

7. return 새로 생성된 사용자 객체 signupResDto를 반환해 성공을 알림

## 2. CreateDto

---

```
export type CreateUserDto = {
  name: string;
  email: string;
  password: string;
  phone: string;
  role: UserRole;
};
```

1. 사용할 객체 export type지정 후 객체의 타입까지 지정.

## 3. SignupResDto

---

```
export type SignupResDto = {
  id: string;
  name: string;
  email: string;
  phone: string;
};
```

1. client 응답 해줄 객체 타입지정

## 4. Service

---

```
 async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepo.findOneByEmail(dto.email);
    if (user) {
      throw new BusinessException(
        'user',
        `${dto.email} already exist`,
        `${dto.email} already exist`,
        HttpStatus.BAD_REQUEST,
      );
    }
    const hashedPassword = await argon2.hash(dto.password);
    return this.userRepo.createUser(dto, hashedPassword);
  }
```

3. async createUser(dto: CreateUserDto): Promise<User> 새사용자를 생성하는 비동기 함수 입력

4. CreateUserDto 형식의 객체 dto를 인수 받고 Promise<User> 를 반환

5. dto.email 주어진 이메일주소로 findOneByEmail 에 해당하는 userRepo에서 사용자를 찾음 데이터 베이스 작업수행

6. 만약 해당 이메일주소로 등록된 사용자가 DB에 있다면(user) BusinessExeption을 throw해라 HttpStatus.BAD_REQUEST를 client 에게

7. hashedPassword 비밀번호 해싱 하는 변수명 지정후
argon2.hash 함수를 사용 (dto.password) 를 해싱해서 hashedPassword 에 할당 [(yarn add argon2)보안강화를 위해사용]

8. 만일 사용자가 존재하지 않는다면, return userRepo, createUser메서드를 호출해 new 사용자 생성(dto, hashedPasseord)라는 인수를 전달해서

## 5. Repo(findOneByEmail)

---

```
  async findOneByEmail(email: string): Promise<User> {
    return this.repo.findOneBy({ email });
  }

  async createUser(dto: CreateUserDto, hashedPassword: string): Promise<User> {
    const user = new User();
    user.name = dto.name;
    user.email = dto.email;
    user.password = hashedPassword;
    user.phone = dto.phone;
    user.role = dto.role;
    return this.repo.save(user);
  }

```

1. 이메일 기반으로 사용자를 찾는 메서드 이다. email이란 문자열 매개변수로 받아 User로 해당주소로 가진 사용자를 찾아 반환한다.

2. createUser 새 사용자를 생성 dto: CreateUserDto, 객체와 hashedPassword 비밀번호를 매개변수로 받는다.

3. const user = new User(); 새로운 객체를 생성 하며,
password는 한번 argon2.hash 하였기에 hashedPassword 이다.

4. return user를 save