# graphql-dotnet-auto-types
An extension of graphql-dotnet ( https://github.com/graphql-dotnet/graphql-dotnet ) that automatically generates InputObjectGraphType and ObjectGraphType starting from Dto classes

Nuget package is here:
https://www.nuget.org/packages/GraphQL.AutoTypes/

Use your specific version related to graphql-dotnet version. So, if you have GraphQL 3.0.0-preview-1648 installed, you need to install GraphQL.AutoTypes 3.0.0.1648 version.


Example:

To generate InputObjectGraphType use this sintax:

    public class TestRequestInputType : GraphQLInputGenericType<TestRequest> { }
    
In this case, our output Dto model is named TestRequest. You need to use convention dto name + "InputType", so in the example we will have TestRequestInputType type


To generate ObjectGraphType use this sintax:

     public class TestResponseType : GraphQLGenericType<TestResponse> { }
    
    
In this case, our input Dto model is named TestResponse. You need to use convention dto name + "Type", so in the example we will have TestResponseType type

In both cases, we can add also other properties to autogenerated types. Example:

    public class TestResponseType : GraphQLGenericType<TestResponse>
    {

        //example if you want add others graphql props to automatic generated props from dto
        public TestResponseType()
        {
            //Computed field example
            Field<StringGraphType>("otherCode", resolve: context => $"{context.Source.Code}-append-other");
        }
    }

PS: enums will be converted into IntGraphType, so I suggest to add a computed field (StringGraphType) or dto output property (string) that makes "ToString()" if you want return enums name too.

Our example Dtos:

    public class TestRequest
    {
        public int Filter1 { get; set; }
        public int Filter2 { get; set; }

    }
    
    public class TestResponse
    {
        public string Code { get; set; }

    }
    
Here ad example mutation, but you can use in queries too:

    FieldAsync<TestResponseType>(
                "demoMutation",
                    arguments: new QueryArguments(new QueryArgument<NonNullGraphType<TestRequestInputType>> { Name = "input" }),
                    resolve: async context =>
                    {

                        //use Netwonsoft deserializer, because default fails if there are other properties outside original type schema; you can create a context extension method to do this
                        TestRequest request = JsonConvert.DeserializeObject(JsonConvert.SerializeObject(context.GetArgument<dynamic>("input")));
			//TestRequest request = context.GetArgument<TestRequest>("input"); //sometimes fails

                        TestResponse response = await _testService.DemoMutation(request);

                        return response;
                    });
 
 Call example:
 
     mutation($input: TestRequestInput!){     
            demoMutation(input: $input){
                code
            }      
     }
     
     {
        "input": {
            "filter1": 100
            "filter2": 200
         }
      }
                
Note that:

1-Mutation field get "TestRequestInputType", that is converted to "TestRequest" request that is passed to service method

2-Service method returns "TestResponse" response, that is converted in TestResponseType

NB: I prefer to use Netwonsoft deserializer, because default fails if there are other properties outside original type schema




You can automatically register all schema types, in startup.cs/ConfigureServices, so you can avoid registering them one by one manually:

    (from t in Assembly.GetAssembly(typeof(YourGraphQLSchema)).GetTypes()
             where t.BaseType.IsGenericType &&
             (t.BaseType.GetGenericTypeDefinition() == typeof(GraphQLGenericType<>) ||
             t.BaseType.GetGenericTypeDefinition() == typeof(GraphQLInputGenericType<>))
             select t)
            .ToList()
            .ForEach(t => services.AddSingleton(t));
            
