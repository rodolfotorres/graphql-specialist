# GraphQL Gateway Guidelines

Guidelines to be followed when developing on the GraphQL gateway. They are categorised as follows:

- :white_check_mark: DO
- :+1: CONSIDER
- :-1: AVOID
- :no_entry_sign: DO NOT

Other resources:

[GraphQL State of Practice](https://sync.hudlnet.com/display/DEV/Technology+-+GraphQL+API)

[GraphQL Engine](https://github.com/graphql-dotnet/graphql-dotnet)

[GraphQL Conventions](https://github.com/graphql-dotnet/conventions)


Index

[Field](#field)

- [Identifier](#identifier)
- [Output](#output)
- [Resolver](#resolver)

[Operation](#Operation)

- [Argument](#argument)
- [Query](#query)
- [Mutation](#mutation)
- [Subscription](#subscription)

[Authentication](#authentication)

[Authorization](#authorization)

[Pagination](#pagination)

[Gateway Structure](#gateway-structure)

[Domain Client Packages](#domain-client-packages)

[Dependency injection](#dependency-injection)

[Data loader](#data-loader)



------



# Field

A unit of data you are asking for in a Schema, which ends up as a field in your JSON response data.

### :+1: CONSIDER - Using `NonNull<T>` on your non-nullable fields

GraphQL types are nullable by default but you can provide a non-null variant with `NonNull<T>`.

```c#
public NonNull<string> Title => Dto.Title;
```

### :+1: CONSIDER - Using describe if the field name is not enough

[GraphQL Conventions](https://github.com/graphql-dotnet/conventions) provides a `Description` attribute that can be used on input or output fields.

##### Input

```c#
public async Task<Connection<Member>> Members(
    [Inject] ITeamService teamService,
    [Inject] IMemberService memberService,
    int? first, int? last, string after, string before,
    [Description("Group id to filter members")] string groupId,
    [Description("Only Disable members")] bool? disabledOnly,
    [Description("Roles to include")] List<string> roles)
```

##### Ouput

```c#
[Description("The gender of the team.")]
public Gender Gender => Dto.Gender;
```



## Identifier

A special field encoded to base64 that allows to have unique Ids on a Graph.

### :white_check_mark: DO - Use `Id.New<T>` from [GraphQL Conventions](https://github.com/graphql-dotnet/conventions) for Id fields

This will create a base64 representation of Type + Id instead of the Id.

```c#
[Description("The graphql Id of the team.")]
public Id Id => Id.New<Team>(Dto.TeamId);
```

### :white_check_mark: DO - Use `IdentifierForType<T>()` to decode GraphQL Ids

[GraphQL Conventions](https://github.com/graphql-dotnet/conventions) provides an easy way to decode GraphQL Ids to be used on internal service calls

```c#
var internalGroupId = groupId.IdentifierForType<Group>();
```

### :+1: CONSIDER - Using GraphQL Ids as input fields

Where not possible, create a migration path for clients by supporting both and deprecating the `InternalId` field.

```c#
[Description("The hudl Id of the team.")]
[Obsolete("Use Id instead")]
public string InternalId => Dto.TeamId;
```

### :-1: AVOID - Exposing internal Ids. (e.g. Mongo ObjectId())

If you need to expose the raw Id (ex: REST call) use `InternalId` property instead of decoding GraphQL Ids on the client.



## Output

Describes an output value for a field.

### :white_check_mark: DO - Map Dtos to a GraphQL types

This will allow you to have well-defined boundaries between your GraphQL layer and your external data providers.

```c#
public Organization Organization => Organization.FromDto(Dto.School);
```

### :+1: CONSIDER - Using `Union<T>` to represent multiple output fields

Union types are very similar to interfaces, but they don't get to specify any common fields between the types.

```c#
[Description("Represents a searchable resource in the video and library domains.")]
public class LibraryContent : Union<VideoItem, PlaylistItem, FileItem>, INode
{
	public Id Id =>
		Id.New<LibraryContent>(DateTime.UtcNow.ToString(CultureInfo.InvariantCulture));
}
```



## Resolver

Functions that determine how a field in your GraphQL schema is computed on the backend.

### :+1: CONSIDER - Using direct mapping between a Dto and a schema field

Prefer direct mapping from a Dto if you can use one otherwise use a method resolver.

```c#
public bool IsSystemGroup => Dto.IsSystemGroup;
```

### :+1: CONSIDER - Using a method resolver when the field is not part of the Dto or you need to pass in arguments

There a few examples where the Dto does not have all the information that we need so we use a function resolver to fetch the information.

```c#
[Description("Group Video Activity")]
public async Task<VideoActivityTypes.VideoActivity> VideoActivity(
	IRootContext context,
	[Inject] IMemberService memberService,
	[Inject] IVideoActivityService videoActivityService)
{
	var memberIds = await memberService.GetMemberIdsForGroup(Dto.GroupId);
	var videoActivityDto = await videoActivityService.GetVideoActivityForMembers(memberIds);
	return VideoActivityTypes.VideoActivity.FromDto(videoActivityDto);
}
```



# Operation

A single query, mutation, or subscription that can be interpreted the GraphQL execution engine.

### :+1: CONSIDER - Grouping your operations on the same domain and type

Group your operations per type and domain. This will help navigate through the schema

Ex: `TeamQuery`, `TeamMutation`, `TeamSubscription`

```c#
public GraphQLSchema(IDependencyInjector dependencyInjector = null)
{
    _requestHandler = RequestHandler
        .New()
        .WithDependencyInjector(dependencyInjector)
        .WithQuery<TeamQuery>()
        .WithMutation<TeamMutation>()
        .WithoutValidation()
        .Generate();
}
```



## Argument

A set of key-value pairs attached to a specific field. Arguments can be literal values or variables.

### :+1: CONSIDER - Using `NonNull<T>` on your non-nullable arguments

This will be picked up by the engine and throw errors before it hits the resolvers.

```c#
[Description("Get invite code by id")]
public async Task<InviteCode> InviteCode(
    [Description("Invite code id")] NonNull<string> inviteCodeId)
```



## Query

A read-only fetch operation to request data from a GraphQL service.

### :+1: CONSIDER - Naming your queries to match the types

Naming things can be hard but your query names should be the same as the Type its quering. There are a few exceptions

```c#
[Description("Get team by id or customer account id")]
public async Task<Team> Team(
    IRootContext context,
    [Description("Team id")] string teamId,
    [Description("Customer account id")] string customerAccountId)
```

```c#
[Description("Get a list of my teams")]
public async Task<List<Team>> MyTeams(
    IRootContext context,
    [Description("Show teams that have one of these membership roles")] List<string> membershipRoles,
    [Description("Show teams that have all of these features")] List<string> requiredFeatures)
```



## Mutation

An operation for creating, modifying and destroying data.

### :+1: CONSIDER - Returning the mutation result after a mutation operation

Allow clients to query on the mutation to save roundtrips to the server.

```c#
public async Task<Group> AddGroup(
    IRootContext context,
    [Description("Group")] NonNull<GroupInput> group,
    [Description("Team id")] NonNull<string> teamId)
```



### :-1: AVOID - Not having a return value after a mutation

GraphQL supports mutations without a return value but that often leads to subsequent requests from the client. It's important to inform clients of the result of the mutation and allow them to ignore it if they choose to do so. 



## Subscription

A real-time GraphQL operation. A Subscription is defined in a schema like queries and mutations.

TBD



# Authentication

TBD



# Authorization

Authorization is a type of business logic that describes whether a given user/session/context has permission to perform an action or see a piece of data.

### :white_check_mark: DO - Use [AuthorizationContextDto](https://github.com/hudl/dotnet-domaintypes-core/blob/56108cd2e80fabd13a000e14c7c88b0be3e2cddd/src/Hudl.DomainTypes.Core/Dto/AuthorizationContextDto.cs)

AuthorizationContextDto holds user info on the call context. This can also be passed as a param to other services

### :no_entry_sign: DO NOT - Put authorization outside the data loader

The Schema authorization happens on the controller level. Authorizing access to data should be done on the `dataloader` or delegated to the service passing the `AuthorizationContextDto` as a param



# Pagination

Most client apis follow the [relay conventions](https://facebook.github.io/relay/graphql/connections.htm)

### :white_check_mark: DO - Use feature-rich pagination with `Connections` for large lists

Connections allows the client to paginate the results of coming from the api.

```c#
public async Task<Connection<Member>> Members(//other inputs
    int? first, int? last, string after, string before)
{
    //...
    return memberDtos
        .Select(Types.Members.Output.Member.FromDto)
        .ToConnection(first, after, last, before, memberDtos?.Count);
}
```

### :+1: CONSIDER - Using `PageInfo` from [GraphQL Conventions](https://github.com/graphql-dotnet/conventions) when working with `Connections`

[GraphQL Conventions](https://github.com/graphql-dotnet/conventions) provides a very handy `PageInfo` that can be used for paginating results on the client.

```c#
[Description("Information about pagination in a connection.")]
public class PageInfo
{
    [Description("When paginating forwards, are there more items?")]
    public bool HasNextPage { get; set; }
    
    [Description("When paginating backwards, are there more items?")]
    public bool HasPreviousPage { get; set; }
    
    [Description("When paginating backwards, the cursor to continue.")]
    public Cursor StartCursor { get; set; }
    
    [Description("When paginating forwards, the cursor to continue.")]
    public Cursor EndCursor { get; set; }
}
```



# Data loader

Data loaders are classes that encapsulate fetching data from a particular service, with built-in support for caching, deduplication, and error handling.

### :white_check_mark: DO - Use the domain dataloaders to fetch data

GraphQL is designed in a way that allows you to write clean code on the server, where every field on every type has a focused single-purpose function for resolving that value. However without additional consideration, a naive GraphQL service could be very "chatty" or repeatedly load data from your databases.

TBD

# Gateway Structure

TBD - Folder and project structure

# Domain Client Packages

TBD - Talk about boundary restrictions

# Dependency injection

### :white_check_mark: DO - Use the attribute `Inject` to pass services to a resolver

This attribute tells [GraphQL Conventions](https://github.com/graphql-dotnet/conventions) to use its dependency injector to resolve service dependencies

```c#
public async Task<Connection<Member>> MembersToRemove(
	[Inject] IMemberService memberService,
	int? first, int? last, string after, string before)
```
