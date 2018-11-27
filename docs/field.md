# Field

A unit of data you are asking for in a Schema, which ends up as a field in your JSON response data.

### :+1: CONSIDER - Using `NonNull<T>` on your non nullable types

GraphQL types are nullable by default but you can provide a non-null variant with `NonNull<T>`.

```c#
public NonNull<string> Title => Dto.Title;
```

### :white_check_mark: DO - Name your fields accordingly

Naming your fields correctly has the end result of creating a self describing schema.

### :+1: CONSIDER - Using describe if the field name is not enough

Conventions provides a `Description` attribute that can be used on input or output fields

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

A special field encoded to base64 that allows to have uniq ids on a Graph.

### :white_check_mark: DO - Use `Id.New<T>` from conventions for Id fields.

This will create a base64 representation of Type + Id instead of the Id.

```c#
[Description("The graphql Id of the team.")]
public Id Id => Id.New<Team>(Dto.TeamId);
```

### :white_check_mark: DO - Use `IdentifierForType<T>()` to decode GraphQL ids

Conventions provides an easy way to decode GraphQL ids to be used on internal service calls

```c#
var internalGroupId = groupId.IdentifierForType<Group>();
```

### :-1: AVOID - Exposing internal ids. (e.g. Mongo ObjectId())

If you need to expose the raw Id (ex: REST call) use `InternalId` property instead of decoding GraphQL ids on the client.

### :no_entry_sign: DO NOT - Decode GraphQL Ids on the client

GraphQL.Conventions should encapsulate how ids are encoded.

### :+1: CONSIDER - Using GraphQL Ids as input fields

Where not possible, create a migration path for clients by supporting both and deprecating the `InternalId` field.

```c#
[Description("The hudl Id of the team.")]
public string InternalId => Dto.TeamId;
```



## Output

Describes an output value for a field.

### :no_entry_sign: DO NOT - Expose Dto's directly. Map it to a GraphQL type

This will allow you to have well define boundaries between your GraphQL layer and your external data providers.

```c#
public Organization Organization => Organization.FromDto(Dto.School);
```

### :+1: CONSIDER - Using `Union<T>` to represent multiple output fields

Conventions provides a `Union<T>`

```c#
[Description("Represents a searchable resource in the video and library domains.")]
public class LibraryContent : Union<VideoItem, PlaylistItem, FileItem>, INode
{
	public Id Id =>
		Id.New<LibraryContent>(DateTime.UtcNow.ToString(CultureInfo.InvariantCulture));
}
```



## Resolver

A function that connects schema fields and types to various backends. Resolvers provide the instructions for turning a GraphQL operation into data. It can retrieve or write data from either an SQL, a No-SQL, graph database, a micro-service or a REST API. Resolvers can also return strings, ints, null, and other primitives.

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
