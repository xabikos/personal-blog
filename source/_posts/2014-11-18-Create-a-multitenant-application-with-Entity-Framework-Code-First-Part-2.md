---
title: Create a multitenant application with Entity Framework Code First - Part 2
date: 2014-11-18 20:15:00
tags: [entity framework, multi-tenant, interceptors, code first]
categories: [multitenant, application design, software as a service]
alias: multitenant/application design/software as a service/2014/11/18/create-a-multitenant-application-with-entity-framework-code-first---part-2.html
---

#### A short introduction

In continuation of [Part 1][part1] about creating a multitenant application with Entity Framework Code First we are going to see how we can use Interceptors to apply filtering when querying the data in a transparent way for our application. It is highly recommended to read the [first part][part1] as this post assumes you are already familiar with the problem.
<!-- more -->
The idea behind this implementation came up based on two very nice projects in Github which try to give a generic solution to the filtering problem. The first project is called [EntityFramework.Filters][effilters] and it was the first one from [Jimmy Bogard][effiltersauthor] which has a NuGet [package][effilterspackage] as well. The second very similar project which solved a couple of issues is [EntityFramework.DynamicFilters][efdynamicfilters] from [jcachat][efdynamicfiltersauthor] which also comes with a NuGet [package][efdynamicfilterspackage]. I have to highlight here that both of the projects give a wider solution to query filtering with Entity Framework and for our solution we are going to use just the general idea of filtering.

***

#### Entity Framework interceptors

[Interceptors][interceptors] is a very powerful mechanism that Entity Framework added in version 6 and allows us to write custom code and then injecting in the framework's execution pipeline. We can use interceptors to extend or modify the functionality of Entity Framework. A very common scenario in to use interceptors for database activity logging as you can see in [this example][interceptors]. For our purposes we will use interceptors to modify first all the query commands to the database and in the next part the insert, update and delete commands.

***

#### The command tree interceptor

The first step is to implement the IDbCommandTreeInterceptor and the TreeCreated method it declares, in order to take control of the query send to the database. The TenantCommandTreeInterceptor class is available below:
{% tabbed_codeblock Command Interceptor%}
    <!-- tab cs -->
public class TenantCommandTreeInterceptor : IDbCommandTreeInterceptor {
    public void TreeCreated(DbCommandTreeInterceptionContext interceptionContext) {
        if (interceptionContext.OriginalResult.DataSpace == DataSpace.SSpace) {
            // Check that there is an authenticated user in this context
            var identity = Thread.CurrentPrincipal.Identity as ClaimsIdentity;
            if (identity == null){
                return;
            }
            var userIdclaim = identity.Claims.SingleOrDefault(c => c.Type == ClaimTypes.NameIdentifier);
            if (userIdclaim == null) {
                return;
            }
            // In case of query command change the query by adding a filtering based on tenantId
            var queryCommand = interceptionContext.Result as DbQueryCommandTree;
            if (queryCommand != null) {
                var newQuery = queryCommand.Query.Accept(new TenantQueryVisitor());
                interceptionContext.Result = new DbQueryCommandTree(
                    queryCommand.MetadataWorkspace,
                    queryCommand.DataSpace,
                    newQuery);
                return;
            }
        }
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

I will try to explain the piece of code even though is not one of easiest concept in .net framework. A [command tree][commandtree] in most cases is the equivalent of an [expression tree][expressiontree] that was created by a LINQ statement and Entity Framework can understand. In our case the query command tree is an object model representation of a query as is explained in the [msdn][commandtree] documentation. If we would like to make it more clear a query command tree is an object representation of a SQL select statement with all required parameters.

Initially we try to get the the identity of the user and verify that is logged in by getting the ClaimTypes.NameIdentifier. If the user is not logged we shouldn't proceed with the interception as our requirement is to filter the data per user. The next few lines of code are relatively simple as we first examine if the command is a query command and if this is the case we continue with the interception. We create the new query which contains the filtering by applying the [Visitor design pattern][visitor] and the next step is to explicitly set the Result property of the interceptionContext by creating a new DbQueryCommandTree and return as the interception has done.

***

#### The TenantQueryVisitor class

Now it's time to see how the Query visitor class looks like.

{% tabbed_codeblock Query expression visitor%}
    <!-- tab cs -->
public class TenantQueryVisitor: DefaultExpressionVisitor {
    public override DbExpression Visit(DbScanExpression expression) {
        var column = TenantAwareAttribute.GetTenantColumnName(expression.Target.ElementType);

        if (!string.IsNullOrEmpty(column)) {
            // Get the current expression
            var dbExpression = base.Visit(expression);
            // Get the current expression binding
            var currentExpressionBinding = DbExpressionBuilder.Bind(dbExpression);
            // Create the variable reference in order to create the property
            var variableReference = DbExpressionBuilder.Variable(currentExpressionBinding.VariableType,
                currentExpressionBinding.VariableName);
            // Create the property based on the variable in order to apply the equality
            var tenantProperty = DbExpressionBuilder.Property(variableReference, column);
            // Create the parameter which is an object representation of a sql parameter.
            // We have to create a parameter and not perform a direct comparison with Equal function for example
            // as this logic is cached per query and called only once
            var tenantParameter = DbExpressionBuilder.Parameter(tenantProperty.Property.TypeUsage,
                TenantAwareAttribute.TenantIdFilterParameterName);
            // Apply the equality between property and parameter.
            var filterExpression = DbExpressionBuilder.Equal(tenantProperty, tenantParameter);
            // Apply the filtering to the initial query
            return DbExpressionBuilder.Filter(currentExpressionBinding, filterExpression);
        }

        return base.Visit(expression);
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

The code here requires some knowledge of entity framework internals but I will try to explain it a bit more even though there are multiple comments in the code. What we are doing here is that we get the initial query and storing it in currentExpressionBinding variable. After doing this we create another expression which the SQL equivalent is "originalQuery AND TenantId = @TenantIdParameter". We could assign the value at this point and don't use a SQL Parameter but this has a major problem as it's highlighted in the comments. Internally Entity Framework caches the command tree by model as explained [here][cache] so if we apply the filtering at this point we will get back always the data for the first user accessed the application something that obvious wrong. This implementation detail adds the need of implementing another interceptor, that is in a lower level, and assign the SQL parameter value we created inside visitor class.

***

#### The command interceptor class

The command interceptor class is shown below:
{% tabbed_codeblock Message POCO%}
    <!-- tab cs -->
internal class TenantCommandInterceptor : IDbCommandInterceptor {
    public void NonQueryExecuting(DbCommand command, DbCommandInterceptionContext<int> interceptionContext){
        SetTenantParameterValue(command);
    }

    public void NonQueryExecuted(DbCommand command, DbCommandInterceptionContext<int> interceptionContext){}

    public void ReaderExecuting(DbCommand command, DbCommandInterceptionContext<DbDataReader> interceptionContext){
        SetTenantParameterValue(command);
    }

    public void ReaderExecuted(DbCommand command, DbCommandInterceptionContext<DbDataReader> interceptionContext){}

    public void ScalarExecuting(DbCommand command, DbCommandInterceptionContext<object> interceptionContext){
        SetTenantParameterValue(command);
    }

    public void ScalarExecuted(DbCommand command, DbCommandInterceptionContext<object> interceptionContext){}

    private static void SetTenantParameterValue(DbCommand command) {
        var identity = Thread.CurrentPrincipal.Identity as ClaimsIdentity;
        if ((command == null) || (command.Parameters.Count == 0) || identity == null) {
            return;
        }
        var userClaim = identity.Claims.SingleOrDefault(c => c.Type == ClaimTypes.NameIdentifier);
        if (userClaim != null) {
            var userId = userClaim.Value;
            // Enumerate all command parameters and assign the correct value in the one we added inside query visitor
            foreach (DbParameter param in command.Parameters) {
                if (param.ParameterName != TenantAwareAttribute.TenantIdFilterParameterName)
                    return;
                param.Value = userId;
            }
        }
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

Here the logic is simpler as we just have to find the SQL Parameter if present and assign the correct value based on the user identity. The IDbCommandInterceptor contains six methods but in our use case we have to intercept only the ones executing before the command is sent to the database. I want to highlight here that is important how we access the required tenantId which in our case is the userId and we can find it from user's identity. Now that async operations are very common this code could potentially executed by a thread that didn't start the entire procedure so we have to make sure we can still access the required info.

***

#### Add interceptors in Entity Framework pipeline

The last step is to make Entity Framework aware of the interceptors we added before. To accomplish this we must use one of the latest addition in Framework the [DbConfiguration][dbconfiguration] class. We have to derive from this class and add the two interceptors as the code snippet below shows:
{% tabbed_codeblock Message POCO%}
    <!-- tab cs -->
public class EntityFrameworkConfiguration : DbConfiguration {
    public EntityFrameworkConfiguration() {
        AddInterceptor(new TenantCommandInterceptor());
        AddInterceptor(new TenantCommandTreeInterceptor());
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

This class is very handy and powerful as we can do several configurations and is discovered automatically by Entity Framework. I want to note here that only one instance of a class deriving from DbConfiguration must exist per AppDomain.


***

This was the second part of a series of three posts. You can find the first part [here][part1]. I have created a [project in Github][github] that contains the full code in a relatively change model. In the next post I will describe how we can use interceptors to modify the insert, update and delete command.

### [Part 1][part1]

### [Part 3][part3]

[part1]: /2014/11/17/Create-a-multitenant-application-with-Entity-Framework-Code-First-Part-1/
[part3]: /2014/11/17/Create-a-multitenant-application-with-Entity-Framework-Code-First-Part-1/
[effilters]: https://github.com/jbogard/EntityFramework.Filters/
[effiltersauthor]: http://lostechies.com/jimmybogard/
[effilterspackage]: https://www.nuget.org/packages/EntityFramework.Filters/
[efdynamicfilters]: https://github.com/jcachat/EntityFramework.DynamicFilters/
[efdynamicfiltersauthor]: https://github.com/jcachat/
[efdynamicfilterspackage]: https://www.nuget.org/packages/EntityFramework.DynamicFilters/
[interceptors]: http://msdn.microsoft.com/en-us/data/dn469464.aspx#BuildingBlocks/
[commandtree]: http://msdn.microsoft.com/en-us/library/vstudio/ee789837(v=vs.100).aspx/
[expressiontree]: http://msdn.microsoft.com/en-us/library/bb397951.aspx/
[visitor]: http://en.wikipedia.org/wiki/Visitor_pattern/
[cache]: https://entityframework.codeplex.com/SourceControl/latest#src/EntityFramework/Infrastructure/Interception/IDbCommandTreeInterceptor.cs
[dbconfiguration]: http://msdn.microsoft.com/en-us/data/jj680699.aspx/
[github]: https://github.com/xabikos/EfMultitenant/