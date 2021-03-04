![](https://images.velog.io/images/ouo_yoonk/post/25fd5b86-a907-4a86-92d9-faccab71da06/character2.gif)

# 뉴편함 📬

항해 99 첫 팀별과제로 만든 간단한 원페이지 서비스!
뉴스레터들을 랜덤하게 8개씩 보여준다. 코멘트, 좋아요를 남길 수 있고, 보기싫은 뉴스레터는 숨길 수 있다. 

[더 자세히 보고싶다면🚀](https://github.com/yooooonk/hanghae99)
## 사용한 스택
- Front-end : vanilla JS, CSS, bootstrap
- Back-end : Flask
- DB : mongoDB

## 상세기능
### JWT 로그인, 회원가입
![](https://images.velog.io/images/ouo_yoonk/post/3fece935-3628-406d-8c2a-fbe637b3ae18/11.gif)
- 회원가입, 로그인 창 노출
- 로그인 여부에 따라 버튼 노출
- jwt 로그인 구현
``` python
@app.route('/index/login', methods=['POST'])
def login():
    
    email = request.form['email']
    password = request.form['password']
	# 로그인한 정보로 db에서 데이터를 가져옵니다
    pw_hash = hashlib.sha256(password.encode('utf-8')).hexdigest()	
    
    result = db.user.find_one({'email': email, 'password': pw_hash})

    # 찾으면 JWT 토큰을 만들어 발급합니다.
    if result is not None:
        # JWT 토큰에는, payload와 시크릿키가 필요합니다.
        # 시크릿키가 있어야 토큰을 디코딩(=풀기) 해서 payload 값을 볼 수 있습니다.
        # 아래에선 id와 exp를 담았습니다. 즉, JWT 토큰을 풀면 유저ID 값을 알 수 있습니다.
        # exp에는 만료시간을 넣어줍니다. 만료시간이 지나면, 시크릿키로 토큰을 풀 때 만료되었다고 에러가 납니다.
        payload = {
            'email': email,
            'exp': datetime.datetime.utcnow() + datetime.timedelta(seconds=60 * 60)
        }
        token = jwt.encode(payload, SECRET_KEY, algorithm='HS256')  # .decode('utf-8')
        print('token', token)
        # token을 줍니다.
        return jsonify({'result': 'success', 'token': token})
    # 찾지 못하면
    else:
        return jsonify({'result': 'fail', 'msg': '아이디/비밀번호가 일치하지 않습니다.'})

```

## 뉴스레터 랜덤 가져오기
![](https://images.velog.io/images/ouo_yoonk/post/db74ccb8-aa51-4f85-9b3e-f5a9405e3112/22.gif)
숨기기한 뉴스레터를 제외하고 랜덤으로 8개를 가져와서 노출한다. jinja2 template을 이용해 __SSR 렌더링__으로 구현했다. 

``` python
@app.route('/')
def home():
    token_receive = request.cookies.get('mytoken')
    print(token_receive)

    try:
        payload = jwt.decode(token_receive, SECRET_KEY, algorithms=['HS256'])
        user_info = db.user.find_one({"email": payload['email']})

        newsletters = list(db.newsletters.aggregate([{'$match': {'title': {'$nin': user_info['hide']}}},
                                                     {'$sample': {'size': 8}},
                                                     {'$project': {'_id': False}}
                                                     ]))

        like = user_info['like']

        if len(like) > 0:
            for letter in newsletters:
                if (letter['title'] in like):
                    letter['like'] = 1

        commentList = user_info['comment'];
        if len(commentList) > 0:
            for letter in newsletters:
                for comment in commentList:
                    if (letter['title'] == comment['title']):
                        letter['comment'] = comment['comment']

        return render_template('index.html', status=user_info, newsletters=newsletters)
    except jwt.ExpiredSignatureError:
        newsletters = list(db.newsletters.aggregate([{'$sample': {'size': 8}}, {'$project': {'_id': False}}]))
        return render_template('index.html', status="expire", newsletters=newsletters)
    except jwt.exceptions.DecodeError:
        newsletters = list(db.newsletters.aggregate([{'$sample': {'size': 8}}, {'$project': {'_id': False}}]))
        return render_template('index.html', newsletters=newsletters)
    # DB에서 저장된 단어 찾아서 HTML에 나타내기
```

``` javascript
    <div class="card-wrapper">
        {% for newsletter in newsletters %}
            <div id="newsletters-{{ newsletter.title_news }}" class="letter" style="size:300px">
                <img class="card-img-top" src="{{ newsletter.image }}" alt="Card image cap">
                <div class="card-body">
                    <a target="_blank" href="{{ newsletter.url }}" class="card-title"
                       id="comment">{{ newsletter.title }}</a>
                    <p class="category">{{ newsletter.category }}</p>
                    <p class="card-text">{{ newsletter.desc }}</p>
                     {% if newsletter.comment %}
                        <ul class="list-group list-group-flush">
                            <li class="list-group-item">{{newsletter.comment}}</li>
                        </ul>
                    {% else %}
                        <div class="md-form">
                        <textarea class="md-textarea form-control" rows="3"
                                  placeholder="코멘트를 입력해주세요"></textarea>
                        <button onclick="comment_insert('{{ newsletter.title }}',this)" type="button"
                                class="btn btn-default">save
                        </button>
                        </div>
                    {% endif %}
                </div>
                <footer class="card-footer">
                    {% if newsletter.like %}
                        <i class="fas fa-star like" onclick="likeLetter('{{ newsletter.title }}',false)"></i>
                    {% else %}
                        <i class="far fa-star dontlike" onclick="likeLetter('{{ newsletter.title }}',true)"></i>
                    {% endif %}
                        <i onclick="deleteLetters('{{ newsletter.title }}')" class="fas fa-ban"></i>
                </footer>
            </div>
        {% endfor %}
    </div>
```

### 좋아요, 코멘트
![](https://images.velog.io/images/ouo_yoonk/post/0696751d-0661-4875-a647-28e8289771f8/33.gif)

