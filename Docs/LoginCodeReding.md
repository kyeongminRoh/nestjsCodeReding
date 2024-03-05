# Login CodeReding

## contorller

```
 @Post('login')
  async login(
    @Req() req,
    @Body() loginReqDto: LoginReqDto,
  ): Promise<LoginResDto> {
    const { ip, method, originalUrl } = req;
    const reqInfo = {
      ip,
      endpoint: `${method} ${originalUrl}`,
      ua: req.headers['user-agent'] || '',
    };

    return this.authService.login(
      loginReqDto.email,
      loginReqDto.password,
      reqInfo,
    );
  }
```

1. @Post 요청으로 /login 앤드포인트로 들어오면 메소드 호출 지정

2. @Req는 req요청객체, @Body로는 loginReqDto로 데이터를 전송 <LoginResDto> 

3. 요청 객체에 ip, method, url 을 추출해 변수에 할당추출된 요청, reqInfo 객체에 ip, 앤드포인트, 사용자 에이전트를 포함함

4. return 으로는 login메서드를 호출 하고 이메일 비밀번호를 인증을 수행하고 reqInfo도 함께 전달.

---

## Service

```
  async login(
    email: string,
    plainPassword: string,
    req: RequestInfo,
  ): Promise<LoginResDto> {
    const user = await this.validateUser(email, plainPassword);
    const payload: TokenPayload = this.createTokenPayload(user.id);

    const [accessToken, refreshToken] = await Promise.all([
      this.createAccessToken(user, payload),
      this.createRefreshToken(user, payload),
    ]);

    const { ip, endpoint, ua } = req;
    await this.accessLogRepository.createAccessLog(user, ua, endpoint, ip);

    return {
      accessToken,
      refreshToken,
      user: {
        id: user.id,
        name: user.name,
        email: user.email,
        phone: user.phone,
      },
    };
  }
```

1. 이메일, 일반비밀번호, 요청 정보 객체들을 Promise LoginResDto 로 반환

2. 제공된 이메일 비밀번호를 사용해사용자를 확인 하는 메서드(user)를 호출

3. 사용자 아이디 기반으로 TokenPayload을 로그인 하면서 주어질 accessToken, refreshToken 을 생성

4. ip, 앤드포인트, (ua)agent 정보를 추출

5. 사용자의 엑세스 로그를 저장한다. 사용자가 어디에서 어떤 앤드포인트에 엑세스 했는지 추적 하기 위함 이다.

6. 성공한 경우 LoginResDto 로 반환

## Repository

```
@Injectable()
export class AccessLogRepository extends Repository<AccessLog> {
  constructor(
    @InjectRepository(AccessLog)
    private readonly repo: Repository<AccessLog>,
    @InjectEntityManager()
    private readonly entityManager: EntityManager,
  ) {
    super(repo.target, repo.manager, repo.queryRunner);
  }

  async createAccessLog(user: User, ua: string, endpoint: string, ip: string) {
    const accessLog = new AccessLog();
    accessLog.user = user;
    accessLog.ua = ua;
    accessLog.endpoint = endpoint;
    accessLog.ip = ip;
    accessLog.accessedAt = new Date(Date.now());
    await this.save(accessLog);
  }
```

1. createAccessLog(User, ua, endpoint, ip) 를 받음 

2. const accessLog 를 새로운 AccessLog()인스턴스를 생성

3. 인자로 받은 user, ua, 앤드포인트, ip, 그리고 new Date인 Date.now 시간을 this.save DB 에 await 저장 완료할때까지.

### TokePayload 토큰은 코드 해석은 다음시간에

```
export type TokenPayload = {
  sub: string;
  iat: number;
  jti: string;
};
```
```
  createTokenPayload(userId: string): TokenPayload {
    return {
      sub: userId,
      iat: Math.floor(Date.now() / 1000),
      jti: uuidv4(),
    };
  }
```
1. sud: 토큰의 주제subject 로 토큰이 발급된 주체를 나타내며 사용자의 고유 식별자로 나타냄
2. iat: issudat으로 토큰이 발급된 시간을 나타냄
3. jti: JWT Id중복사용을 방지하기 위해 사용

```
  async createAccessToken(user: User, payload: TokenPayload): Promise<string> {
    const expiresIn = this.configService.get<string>('ACCESS_TOKEN_EXPIRY');
    const token = this.jwtService.sign(payload, { expiresIn });
    const expiresAt = this.calculateExpiry(expiresIn);

    await this.accessTokenRepository.saveAccessToken(
      payload.jti,
      user,
      token,
      expiresAt,
    );

    return token;
  }
```

1. user: 액세스 토큰이 발급될 사용자 payload: 액세스토큰이 포함될 정보를 담은 페이로드

2. 