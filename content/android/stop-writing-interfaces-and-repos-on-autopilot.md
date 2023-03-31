---
title: "Stop Writing Interfaces and Repositories on Autopilot
summary: "The best practice of writing repositories and interfaces has been baked into my mind ever since I started with Android development. Recently, I learned that this can backfire if you don't really need them"
date: 2023-03-30T19:37:55+02:00
draft: true
---

The best practice of writing repositories and interfaces has been baked into my mind ever since I started with Android. Recently, I learned that this can backfire if you don't really need them.

## The problem
Let's say we need to create a screen where the user can see and edit his online profile. No caching, no tests. Sounds simple enough, let's start coding.

First, we need a way for our ViewModel to get the current profile data so that we can display it. We don't want it to know anything about the way this is done, and we're using an architecture based on use cases, so we create a use case:

```kt
class GetProfile(private val profileRepository: ProfileRepository) {
    suspend operator fun invoke(): Profile = profileRepository.getProfile()
}

data class Profile(val name: String)
```

The use case needs to get the data from somewhere. "Get data from somewhere", you say? Sounds like we need a repository: 

```kt
interface ProfileRepository {
    suspend fun getProfile(): Profile
}

class ApiProfileRepository(private val api: Api) : ProfileRepository {
    override suspend fun getProfile(): Profile = api.getProfile()
}
```

The repository is defined by an interface. Its implementation gets the Profile entity from our API. The ViewModel knows nothing of the repository. The use case knows nothing of the API. Each of these layers can be independently changed as long as they preserve their public interfaces. All is well.

Now, let's edit the user's name. Once again, we create a use case: 

```kt
class EditName(private val profileRepository: ProfileRepository) {
    suspend operator fun invoke(newName: String): Boolean = profileRepository.editName(newName)
}
```

The use case needs to perform this somehow. We already have a profile repository which is loosely defined as the place where profile operations occur, so let's add another method: 

```kt
interface ProfileRepository {
    suspend fun getProfile(): Profile
    suspend fun editName(newName: String): Boolean
}

class ApiProfileRepository(private val api: Api) : ProfileRepository {
    override suspend fun getProfile(): Profile = api.getProfile()
    override suspend fun editName(newName: String): Boolean = api.editName(newName)
}
```

The implementation just returns whether or not the API succeeded in editing the name. The layers are still cleanly separated and independent. Our use cases are ready to be used to implement the profile screen. What's the problem?

## Why the interface?
We will only get profile data from the API and we don't have any tests at the moment. This means that we will only have one implementation of `ProfileRepository`, and its only purpose is to hide the API. But do we really need an interface for this?

```kt
class ProfileRepository(private val api: Api) {
    suspend fun getProfile(): Profile = api.getProfile()
    suspend fun editName(newName: String): Boolean = api.editName(newName)
}
```

By removing the interface and renaming the class, we have preserved abstraction and reduced complexity. The caller still doesn't know we're using an API. During maintenance, developers will be taken straight to the implementation, which saves time and energy. Besides, we can easily create an interface from a class' public methods when the need for testing or polymorphism arises.

### Point #1
Don't write interfaces before you need to. Classes can abstract implementation details on their own.

## Why the repository? 
Our repository is now simpler, but what is its purpose? It just delegates its operations to the API and hides it. Now, this does have its benefits. If something about the API changes, we can react to it in the repository and not touch the use cases. This is a clean approach, as the use case shouldn't know about low-level implementation details, like the network. However, with this app's requirements, I would argue it's better to bend clean architecture in favor of simplicity.

```kt
class GetProfile(private val api: Api) {
    suspend operator fun invoke(): Profile = api.getProfile()
}

class EditName(private val api: Api) {
    suspend operator fun invoke(newName: String): Boolean = api.editName(newName)
}
```

Future developers now have one fewer abstraction layer to think about during maintenance. The API is still hidden from the presentation layer and it can be controlled from the use case. 

### Point #2
Don't write repositories if they just delegate their calls to another class. Remove the middleman and use the class directly.

## When should I use interfaces, then?
Use them when you need multiple implementations behind the same interface. Let's say we need to get a user's GPS location. There are multiple ways to do this on Android: 

1. Fused Location Provider for devices with Google Mobile Services
2. Location Kit for devices with Huawei Mobile Services
3. Location Manager for devices with neither

Then we could have something like this:

```kt
interface LocationProvider {
    suspend fun getCurrentLocation(): Coordinates
}

class GmsLocationProvider(...) : LocationProvider {...}
class HmsLocationProvider(...) : LocationProvider {...}
class LocationManagerLocationProvider(...) : LocationProvider {...}
```

Now every class that needs to get the current location will use the `LocationProvider` interface which is injected based on the capabilities of the current device. This can be done with your dependency-injection library of choice, or even manually. If we're using Dagger: 

```kt
@Module
class LocationProviderModule {
    @Provides
    fun provideLocationProvider(): LocationProvider = 
        when {
            hasGoogleMobileServices() -> GmsLocationProvider()
            hasHuaweiMobileServices() -> HmsLocationProvider()
            else -> LocationManagerLocationProvider()
        }
}
```

## And repositories?
Use them when you need to coordinate multiple data sources. Let's say we need to fetch a news feed and cache it for offline viewing. Then a repository is justified and we could have something like this:

```kt
class NewsFeedRepository(
    private val api: Api,
    private val database: Database
) {
    suspend fun getNewsFeed(): NewsFeed? {
        api.getNewsFeed()?.let { database.insertOrUpdateNewsFeed(it) }
        return database.getNewsFeed()
    }
}
```

Now every class that needs to get a news feed will use the `NewsFeedRepository` and it won't have to worry about the caching logic. In fact, it won't even know it's happening, which is exactly what repositories are for. 

## Conclusion
Before you make an interface or a repository, think if they will be used for their actual purpose. If not, save yourself and your team the trouble and just exclude them. In other words, remember YAGNI and KISS those interfaces and repositories goodbye :)