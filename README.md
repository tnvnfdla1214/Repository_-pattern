데이터 출처(로컬 DB인지 API응답인지 등)와 관계 없이 동일 인터페이스로 데이터에 접속할 수 있도록 만드는 것을 Repository 패턴이라고 합니다. 레포지토리는 데이터 소스에 액세스하는 데 필요한 논리를 캡슐화하는 클래스 또는 구성 요소입니다.

아래 이미지를 보면 레포지토리를 잘 이해할 수 있습니다. Presentation 레이어(View, ViewModel)나 Domain 레이어에서 Data 레이어(Repository, Model 등)에 접근할 때, 데이터 소스의 위치(서버, Local DB)를 몰라도 일관성 있게 원하는 데이터를 취할 수 있도록 돕는 것이 Repository의 역할입니다.

 <img src = "https://user-images.githubusercontent.com/48902047/147075209-b00acafc-75f0-4d51-b7e8-fb8cdfb368dd.png" width="50%" height="50%">
 
## Repository Pattern이란
“저장소”라는 뜻을 가진 Repository는 데이터 소스 레이어와 비즈니스 레이어 사이를 중재하는 역할을 합니다. 데이터 소스에 쿼리를 날리거나 데이터를 다른 domain에서 사용할 수 있도록 새롭게 매핑할 수 있습니다.

 <img src = "https://user-images.githubusercontent.com/48902047/147231453-f33cb801-b8d8-4b95-9e66-457d887a5c1f.png" width="50%" height="50%">
 
데이터가 있는 저장소인 RemoteDataSource와 LocalDataSource를 추상화하여 중앙 집중 처리 방식을 구현합니다.

데이터를 사용하는 도메인에서 비즈니스 로직에만 집중할 수 있습니다. ex) ViewModel에서는 데이터가 Local DB에서 오는지, 서버 API를 통해 오는지 알 필요가 없다. Repository를 참조해 제공하는 데이터를 이용하기만 하면 됩니다.

Repository가 추상화되어 있으므로, 항상 같은 Interface로 데이터를 요청할 수 있습니다.


## 구현

기본적으로 프로젝트 폴더 내에 *data*라는 Package 경로에 mapper,model,repository 디렉토리를 생성하여 관리 합니다.

<img src = "https://user-images.githubusercontent.com/48902047/147247494-206491e9-5c8a-4e89-bb23-1baa7b14c4c2.png" width="50%" height="50%">

### Mapper
```kotlin
//Mapper.kr
import com.mtjin.androidarchitecturestudy.data.model.search.MovieEntity
import com.mtjin.androidarchitecturestudy.domain.model.search.Movie
//Movie <- MovieEntity
fun mapperToMovie(movies: List<MovieEntity>): List<Movie> {
    return movies.toList().map {
        Movie(
            it.actor,
            it.director,
            it.image,
            it.link,
            it.pubDate,
            it.subtitle,
            it.title,
            it.userRating
        )
    }
}
//MovieEntity <- Movie
fun mapperToMovieEntity(movies: List<Movie>): List<MovieEntity> {
    return movies.toList().map {
        MovieEntity(
            it.actor,
            it.director,
            it.image,
            it.link,
            it.pubDate,
            it.subtitle,
            it.title,
            it.userRating
        )
    }
}
```
위의 Mapper는 Entity와 Model사이 바꿔주는 함수입니다. 이 함수를 만듦으로서 BD변경시 Mapper로 정의한 클래스와 데이터를 가져오던 Repository만 변경하면 되는 장점이 있습니다.
















### 
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

