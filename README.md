# 레파지토리 패턴이란

데이터 출처(로컬 DB인지 API응답인지 등)와 관계 없이 동일 인터페이스로 데이터에 접속할 수 있도록 만드는 것을 Repository 패턴이라고 합니다. 레포지토리는 데이터 소스에 액세스하는 데 필요한 논리를 캡슐화하는 클래스 또는 구성 요소입니다.

아래 이미지를 보면 레포지토리를 잘 이해할 수 있습니다. Presentation 레이어(View, ViewModel)나 Domain 레이어에서 Data 레이어(Repository, Model 등)에 접근할 때, 데이터 소스의 위치(서버, Local DB)를 몰라도 일관성 있게 원하는 데이터를 취할 수 있도록 돕는 것이 Repository의 역할입니다.

 <img src = "https://user-images.githubusercontent.com/48902047/147075209-b00acafc-75f0-4d51-b7e8-fb8cdfb368dd.png" width="50%" height="50%">
 
 보통 레포지토리를 언급할 때, 캡슐화라는 용어가 등장합니다. 이는 객체의 속성과 행위(함수)를 하나로 묶고 구현 내용의 일부 혹은 전체를 외부로부터 감추는 것을 얘기합니다. 이 경우에는 데이터 레이어의 소스들이 캡슐화 대상입니다.

Repository라는 인터페이스만 ViewModel이 접근합니다. 그렇게 함으로서 레포지토리 너머의 데이터 소스가 추가 혹은 제거되는 변경이 있더라도 도메인 레이어나 프레젠트 레이어는 알 수가 없습니다. 캡슐화가 되었기 때문. 동시에 알 필요도 없습니다. 변경에 따른 작업은 레포지토리와 레포지토리 구현체에만 있기 때문입니다. Repository가 없다면 Model 따로 Remote Data Source 따로 처리를 해줘야 했을 것입니다. 이 두 가지(서버, 로컬) 데이터 소스를 동시에 사용하는 경우라면 더욱 작업량이 늘었을 것입니다. 그래서 개발자는 Repository가 없을 때보다 있을 때 데이터 소스가 변경되는데 부담을 적게 느끼게 됩니다.

이렇게 레포지토리로 인한 중간 단계의 추상화로 모듈화가 명확해지고 유지보수성과 단위테스트 검증이 쉬워집니다.

## 구현

기본적으로 프로젝트 폴더 내에 *data*라는 Package 경로를 생성하여 활용합니다.

### Interface 구성
```kotlin
//UserRepository.kt
interface UserRepository {
    //POST	회원가입
    suspend fun postSignUp(body: ReqSignUp): ResSignUp

    //POST	로그인
    suspend fun postSignIn(body : ReqSignIn): ResSignIn
}
```
### Impl 구현
```kotlin
//UserRepositoyrImpl.kt
class UserRepositoryImpl(private val remoteDataSource : UserDataSource) : UserRepository {
    override suspend fun postSignUp(body: ReqSignUp): ResSignUp = remoteDataSource.postSignUp(body)

    override suspend fun postSignIn(body: ReqSignIn): ResSignIn = remoteDataSource.postSignIn(body)
}
```
### model 구현
```kotlin
//ReqSignIn.kt
data class ReqSignIn(
    // 로그인
    val email: String,
    val password: String
)
//ReqSignUp.kt
data class ReqSignUp(
    // 회원가입
    val email: String,
    val password: String
)
```
### 의존성 주입
```kotlin
//RepositoryModule.kt
...
single<UserRepository> { UserRepositoryImpl(get()) }
...
```
### ViewModel

```kotlin
class UserViewModel(private val userRepository: UserRepository) : ViewModel() {

    .
    .
    fun postSignUp() = viewModelScope.launch {
        runCatching {
            userRepository.postSignUp(
                ReqSignUp(
                    requireNotNull(_idData.value),
                    requireNotNull(_passData.value)
                )
            )
        }
            .onSuccess {
                _signUpEvent.postValue(Event(true))
            }
            .onFailure {
                _signUpEvent.postValue(Event(false))
                it.printStackTrace()
            }
    }

    fun postSignIn() = viewModelScope.launch {
        kotlin.runCatching {
            userRepository.postSignIn(
                ReqSignIn(
                    requireNotNull(_idLogin.value),
                    requireNotNull(_pwdLogin.value)
                )
            )
        }
            .onSuccess {
                _signInEvent.postValue(Event(true))
                _userId = it.data.userId
                _petId = it.data.petId
            }
            .onFailure {
                _signInEvent.postValue(Event(false))
                it.printStackTrace()
            }
    }
    .
    .
}
```

