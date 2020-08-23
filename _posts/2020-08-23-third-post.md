---
title: "안녕, Passport"
tags: [Node.js, Passport]
author: 박지환
---

안녕하세요. 우아한 테크 캠프 3기에 참여하고 있는 박지환입니다.

저는 글쓰기 실력이 나빠서 평소 글을 쓰는 편은 아닌데요 읽어주는 사람들이 있는 좋은 기회에 용기내서 한번 써보려고 합니다.

어떤 주제로 글을 써야 고민을 많이하고 글을 쓰다 지우고 했는데요 (지금이 4번째입니다 ㅎㅎ..) 결국 제가 이번 프로젝트와 관련해서 개발 관련 글을 써보려고 합니다.

# 안녕 :cry:, Passport

이번 글에서는 Passport에 대해 얘기해보려고 해요.

저번 프로젝트의 요구 사항 중에 Passport가 있었고 Passport가 로그인 관련 로직을 구현할때 편하기 때문에 많이들 사용하셨을 텐데요.

저는 많은 고민 끝에 이번 프로젝트에서 Passport를 사용하지 않기로 결정했습니다.

그럼 제가 어떤 고민을 했는지 알아볼까요?

## Passport?

먼저, [Passport](http://www.passportjs.org/)는 Node.js의 인증 관련 미들웨어 입니다.

Passport는 다양한 로그인 관련 로직을 처리하기 위해서 상황 맞는 Strategy를 제공하고 있는데요 Strategy는 독립적인 module로 제공되고 있습니다. 따라서, 사용자는 상황에 맞게 필요한 Strategy만 선택해서 사용할 수 있는 거죠.

username과 password를 통해 인증을 처리하는 **local Strategy**를 예로 들어보겠습니다.

아까 Strategy는 독립적인 module로 제공된다고 했죠? 따라서, local Strategy를 먼저 설치해줍니다.(passport는 깔려있다는 전제)

```bash
$ npm i passport-local
```

다음으로 해당 Strategy을 설정해줍니다.

```js
var passport = require("passport"),
  LocalStrategy = require("passport-local").Strategy;

passport.use(
  new LocalStrategy(function (username, password, done) {
    User.findOne({ username: username }, function (err, user) {
      if (err) {
        return done(err);
      }
      if (!user) {
        return done(null, false, { message: "Incorrect username." });
      }
      if (!user.validPassword(password)) {
        return done(null, false, { message: "Incorrect password." });
      }
      return done(null, user);
    });
  })
);
```

> 위의 예제는 공식 문서에서 가져온 예제로 MongoDB를 기준으로 작성되었습니다.

위 예시에서 done을 verify Callback이라고 부르는데 첫번째 인자로 err, 두번째 인자로 정보, 세번째 인자로 flash message(생략 가능)가 들어갑니다.

```js
// 세번째 인자로 메시지를 전달해줄 수도 있다.
// flash message라고 부른다
return done(null, false, { message: "비밀번호가 맞지 않습니다" });
```

로그인에 성공하면 passport는 session을 통해 해당 유저를 serialize시켜요. 해당 유저를 기록한다고 이해하면 될 것 같습니다.

```js
passport.serializeUser(function (user, done) {
  done(null, user.id);
});
```

verify callback의 두번째 인자가 serializeUser의 콜백 함수의 첫번째 인자로 들어가요.(요부분이 조금 어렵습니다)

즉, 로그인 성공시 위에서 done(null, **user**)을 실행했고 여기서 user가 function(**user**, done)의 user로 들어가요.

```js
done(null, user) -> function(user, done)
```

serializeUser가 하는 일은 user 정보를 받고 해당 user 정보를 session에 기록하는 일입니다.(즉, 로그인시 한번 실행되요)

위의 예제에서는 user의 id를 session에 저장하고 있어요.

passport는 기본적으로 세션을 위해 express-session을 사용하고 있기 때문에 user id에 해당하는 sessionId를 생성하고 cookie로 클라이언트에게 내려줍니다. (express-session이 cookie를 사용)

## 정리하면 로그인에 성공할시 serializeUser 함수가 user 정보를 session에 저장하고 cookie를 보낸다!

```js
passport.deserializeUser(function (id, done) {
  User.findById(id, function (err, user) {
    done(err, user);
  });
});
```

로그인 이후에는 요청에는 deserializeUser가 실행되는데 해당 함수는 쿠키에 있는 sessionId를 통해 User의 id를 찾고 이를 통해 해당 유저를 DB에서 찾은 다음 req.user에 정보를 담아 보냅니다.

그러면 다음 미들웨어는 req.user를 통해 인증 여부를 확인할 수 있겠죠?

여기까지 passport의 전반적인 설명과 passport가 제공해주는 serializeUser, deserializeUser 기능을 설명했는데요 passport가 제공해주는 serializeUser 기능을 사용하고 싶으면 express-session을 사용해야하며 서비스를 위해서는 session-storage도 설정해주는 것이 좋습니다. (지금은 메모리에 session 저장)

실제로 데이터를 볼까요?

- 클라이언트 쿠키

```js
connect.sid=s%3AWKNdgG0i8zvN-dNJrvjsg6rRAuFBtCcf.JID7xRCHC0%2BqPHnKKSaZO%2FT7cgFAEG7oVB3rLSncQzM; Path=/; Domain=localhost; HttpOnly;
```

sessionId가 잘 담겨 있습니다.

- req.session

```js
Session {
  cookie: { path: '/', _expires: null, originalMaxAge: null, httpOnly: true },
  passport: { user: 1 }
}
```

express-session이 파싱해준 결과인데 passport 객체 안에 user id가 잘 담겨있습니다.

- req.user

```js
{
  id: 1,
  username: 'jack',
  password: 'secret',
  displayName: 'Jack',
  emails: [ { value: 'jack@example.com' } ]
}
```

deserializeUser가 찾아서 넘겨준 req.user로 유저 정보가 잘 담겨있습니다.

## 고민거리?

지금까지 passport에 대해 알아봤는데요 자세한 내용은 공식 문서를 확인하시면 될 것 같습니다.

지금부터는 제가 고민했던 내용에 대해 얘기해보겠습니다.

항상 라이브러리에 대한 고민은 쓰면 편한데 현재 내 상황에 딱 맞는 라이브러리는 없고 커스터마이징이 어렵다는 점인 것 같습니다.

저의 경우도 그랬습니다.

1. **세션을 사용하지 않는다**

   - 이번 프로젝트에서는 JWT를 사용하여 서버에서 세션 정보를 가지고 있지 않는다는 요구 사항이 있었습니다. 따라서, express-session을 사용하지 않는다는 뜻이고, Passport가 제공해주는 serializeUser, deserializeUser 기능을 사용하지 못한다는 뜻이었습니다.
   - Passport를 사용하는 이유가 편하기 때문인데 serializeUser, deserializeUser를 사용하지 못한다면 Passport를 사용하는 의미가 없다고 생각했습니다.

2. **OAuth 로그인만 사용한다**

   - 저희 팀은 이번 프로젝트에서 OAuth 로그인만 사용하기로 했습니다. Passport가 OAuth 로그인을 쉽게 할 수 있도록 OAuth 전략을 제공해주고 있지만 OAuth를 사용하기 위해서는 두개의 router가 필요하고 SPA 개발과도 잘 맞지 않았습니다.

   - 예를 들어, passport를 사용하는 경우 **로그인**을 누르면 server로 요청이가고 서버에서 인증 화면을 띄워주도록 설계되어있습니다. 인증을 하면 /callback router로 다시한번 요청가 인증 로직을 마무리합니다.

   - 이는 SPA 개발에 적합한 구조가 아니라고 생각했고 router가 많아지는 단점이 있다고 판단했습니다.

     ```js
     // OAuth Strategy 설정
     passport.use('provider', new OAuth2Strategy({
         authorizationURL: 'https://www.provider.com/oauth2/authorize',
         tokenURL: 'https://www.provider.com/oauth2/token',
         clientID: '123-456-789',
         clientSecret: 'shhh-its-a-secret'
         callbackURL: 'https://www.example.com/auth/provider/callback'
       },
       function(accessToken, refreshToken, profile, done) {
         User.findOrCreate(..., function(err, user) {
           done(err, user);
         });
       }
     ));

     // 두개 router 필요
     app.get('/auth/provider', passport.authenticate('provider'));
     app.get('/auth/provider/callback',
       passport.authenticate('provider', { successRedirect: '/',
                                           failureRedirect: '/login' }));
     ```

3) **모듈 설치**

   - OAuth 로그인을 구현하기 위해서 passport와 passport-oauth를 설치해야되는데 OAuth 로그인만 하는데 이는 필요 이상이라고 느꼈습니다.

## 결론

사실 위의 고민은 크게 중요하지는 않죠. 이미 passport로 OAuth 로그인을 구현해놓은 상태였고 계속 진행을 해도 프로젝트를 진행하는데는 아무 문제가 없었을 거에요.

오히려 이미 해놓은 코드를 뒤집고 제가 처음부터 짜여되니 시간이 더 걸리겠죠.

예전의 저라면 그냥 진행했을 것 같아요.

하지만 우아한 테크 캠프에 와서 **Honux**님과 **크롱**님의 집요한(^^) 리뷰, 저번에 들었던 **김민태**님의 강연등으로 "이게 맞는건가?", "지금 상황에 적합한 라이브러리인가?"에 대해 계속 고민하게 되었고 결국 passport를 제거하게 되었습니다! :smile:

```js
router.get(
  "/api/github-login",
  getGithubUser,
  findOrCreateUser,
  async (req: Request, res: Response) => {
    const user = req.user;
    const token = await encodeJwt(user);
    res.status(200).json({
      token,
    });
  }
);
```

제가 직접 구현한 코드인데요 Passport 때와 달리 하나의 router에서 처리 가능하고 여러개의 middleware를 사용하니 Passport와 달리 로직 흐름을 한눈에 알 수 있었습니다.

Passport를 사용할 때는 verify callback의 두번째 인자의 serializeUser 콜백 함수의 첫번째 인자로 들어가는 등 따라가기 힘든 부분이 많았는데요 제가 직접 구현하니 수정도 쉽고 읽기도 쉬웠습니다.

또한, 인증창을 Front에서 띄워주니 React에서 로그인 관련 로직을 처리하기 더 수월했습니다.

## 느낀점

감사하게도 팀원들이 동의해주셔서 라이브러리 없이 직접 OAuth를 구현해보았는데요 이것이 좋은 결정이든 아니든 이런 고민과 과정이 이번 우아한 테크 캠프에서 얻은 것이 아닌가 싶습니다.

예전에는 라이브러리가 정답이라 믿고 어떻게든 라이브러리를 사용하면서 정답을 찾으려고 했는데요 이제는 라이브러리는 ''그냥 남이 만든 코드야''라는 자신감이 생긴 것 같습니다.

그래서 상황에 맞지 않거나 too much:moneybag: 하다고 생각이 들면 "내가 직접 해볼까?"라고 생각할 수 있는거죠.

이게 정답인가? 현재 상황에 맞는가?에 대해 끊임없이 고민하고 생각하는 과정을 이번 우아한 테크 캠프를 통해 배운 것 같습니다.

여러분들도 현재 진행하는 프로젝트에 맞지 않는 라이브러리를 쓰고 있다고 생각하면 한번 직접 만들어보세요! :desktop_computer:

## + 안녕​ :smile:, query-parser

추가로 frontend와 backend가 같이 사용하는 함수가 있었는데 이를 npm으로 배포해서 같이 사용하면 좋겠다라고 생각해서 query-parser라는 이름으로 배포했습니다. (프로젝트에서만 쓸거여서 readme는 대충 만들었어요. 봐주세요 :sob:)

해당 모듈은 Query String을 객체로 변환해주는 모듈이에요.

npm 배포 엄청 어렵게 생각했는데 막상 해보니 별거 없네요.

여러분들도 npm에 배포해보는 경험을 해보면 좋을 것 같습니다!

현재 상황에 맞게 나만의 라이브러리를 만들고 사용하는 과정 또한 이번 우아한 테크 캠프에서 배웠다고 생각해요.

앞으로도 계속 고민하고 생각하는 개발자가 되고 싶습니다.

## 글을 마치며..

이번주 금요일이면 벌써 우아한 테크 캠프도 끝이 나네요 :cry:

남은 기간 화이팅하셔서 데모때 멋진 결과물 보여주세요. 화이팅~!! :punch:

(혹시 다 읽으신 분 있다면 정말 감사합니다.. :bow:)
