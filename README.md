===========
HAL Library
===========

This HAL-library is a Java API to help build a REST API according to [HAL
specification][]

This library aims at easing the bookkeeping around [relations][] and [profiles][]


[HAL specification]: http://stateless.co/hal_specification.html
[relations]: https://tools.ietf.org/html/draft-kelly-json-hal-07#section-8.2
[profile]: https://tools.ietf.org/html/rfc6906
[profiles]: https://tools.ietf.org/html/rfc6906


Usage
=====

The main classes in this solution are both `HalRepresentation<T>` and
`HalCollectionRepresentation<T>`. The `HalRepresentaiton` represents a
single HAL resource of a specific [profile][], whereas - you guessed it
- the `HalCollectionRepresentation` represents a collection of HAL
resources (albeit all with the same [profile][]).


Example
-------

Lets consider an API where you can get information about a certain user
via a GET request to `https://api.example.com/user/1`.

Below would be an implementation against the current work-in-progress.


```java

class UserDto {
	@JsonProperty String username;
	@JsonProperty String email;

	public UserDto(User user) {
		this.username = user.getUsername();
		this.email = user.getEmail();
	}
}


@Path(UserResource.PATH)
class UserResource {
	public static final String PATH = "/user/{id}";

	// These would be kept in a central place.
	// TODO let the HAL library keep track of relations
	public static final String SUBSCRIPTIONS = "http://relation.example.com/subscription";
	public static final String USER_AVATAR = "http://relation.example.com/user-avatar";

	public Response get(Long userId) {
		User user = userRepository.getId(userId);
		return Response.ok(createRepresentation(user)).build();
	}

	public HalRepresentation<UserDto> createRepresentation(User user) {
		UserDto dto = new UserDto(user);
		HalRepresentation<UserDto> representation = new HalRepresentation<UserDto>(dto, getUri(user));


		representation
				// This is a fixed link
				.setLink(SUBSCRIPTIONS, userSubscriptionsResource.getUri(user))
				// This will generate a templated link, where the API consumer
				// can fill in query-parameters
				.setTemplatedLink(USER_AVATAR, userAvatarResource.getUriTemplate(user));

		for (Address address: user.getAddresses()) {
			representation
					// This will generate a list of links
					.addLink(ADDRESS, addressResource.getUri(address))
					// This will embed the address resource
					.addResource(ADDRESS, addressResource.createRepresentation(address));
		}

		return representation;
	}

	public URI getUri(@Nonnull User user) {
		// This would be abstracted away in a common context service
		UriBuilder builder = UriBuilder.fromUri("https://api.example.com/");

		// Get template from @Path annotation
		builder = builder.path(getClass());

		return builder.build(user.getId());
	}


}
```
