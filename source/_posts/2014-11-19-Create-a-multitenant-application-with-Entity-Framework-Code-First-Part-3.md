---
title: Create a multitenant application with Entity Framework Code First - Part 3
date: 2014-11-19 20:15:00
tags: [entity framework, multi-tenant, interceptors, code first]
categories: [multitenant, application design, software as a service]
alias: multitenant/application design/software as a service/2014/11/19/create-a-multitenant-application-with-entity-framework-code-first---part-3.html
keywords:
- entity framework
- multitenant
- code first
---

#### Connection with previous posts

This is the last post of the series of how we can use Entity Framework Code First to create a multitenant application. You are requested first to read [Part 1][part1] where there is an introduction in the problem we are trying to solve and some infrastructure code required to continue. [Part 2][part2] describes the query filtering that is happening automatically in the entire application. In this post I am going to show how we can use the CommandTree interceptor in order to modify the insert, update and delete commands. The idea of implement something like this came to my mind after watching [Rowan Miller's][miller] excellent [session][session] in North America TechEd, which I highly recommend you to watch.
<!-- more -->
***

#### Modification of insert command

Let's start by presenting the code of insert command which is probably the simplest case. What we want to achieve is to always assign the correct TenantId when saving an entity that has this property. I have to remind here that Message class has a private set in TenantId property so there is no way to assign it from the code base.
{% tabbed_codeblock Message POCO%}
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
            var userId = userIdclaim.Value;

            var insertCommand = interceptionContext.Result as DbInsertCommandTree;
            if (insertCommand != null) {
                var column = TenantAwareAttribute.GetTenantColumnName(insertCommand.Target.VariableType.EdmType);
                if (!string.IsNullOrEmpty(column)) {
                    // Create the variable reference in order to create the property
                    var variableReference = DbExpressionBuilder.Variable(insertCommand.Target.VariableType,
                        insertCommand.Target.VariableName);
                    // Create the property to which will assign the correct value
                    var tenantProperty = DbExpressionBuilder.Property(variableReference, column);
                    // Create the set clause, object representation of sql insert command
                    var tenantSetClause =
                        DbExpressionBuilder.SetClause(tenantProperty, DbExpression.FromString(userId));

                    // Remove potential assignment of tenantId for extra safety
                    var filteredSetClauses =
                        insertCommand.SetClauses.Cast<DbSetClause>()
                            .Where(sc => ((DbPropertyExpression)sc.Property).Property.Name != column);

                    // Construct the final clauses, object representation of sql insert command values
                    var finalSetClauses =
                        new ReadOnlyCollection<DbModificationClause>
                        (new List<DbModificationClause>(filteredSetClauses) {
                            tenantSetClause
                        });

                    var newInsertCommand = new DbInsertCommandTree(
                        insertCommand.MetadataWorkspace,
                        insertCommand.DataSpace,
                        insertCommand.Target,
                        finalSetClauses,
                        insertCommand.Returning);

                    interceptionContext.Result = newInsertCommand;
                    return;
                }
            }
        }
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

The most important piece of code is where we create the set clause. To further explain this is an object representation of a SQL pair values like INSERT INTO TABLE (TenantId) VALUE (the value of tenantId). After doing this we filter the original collection of set clause and remove any possible existence of the same set clause. As a final step we create a new insert command by assigning the original values and replacing only the set clause. The interception ends by setting the result explicitly.

***

#### Modification of update command

Now we have to modify any update command that is sent to the database and remove any change in the value of tenantId and add an extra where statement based on the tenantId. So after the interception any SQL update command is going to have an extra and statement like AND TenantId = 'value of tenantId'.
{% tabbed_codeblock Message POCO%}
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
            var userId = userIdclaim.Value;
            var updateCommand = interceptionContext.Result as DbUpdateCommandTree;
            if (updateCommand != null) {
                var column = TenantAwareAttribute.GetTenantColumnName(updateCommand.Target.VariableType.EdmType);
                if (!string.IsNullOrEmpty(column)){
                    // Create the variable reference in order to create the property
                    var variableReference = DbExpressionBuilder.Variable(updateCommand.Target.VariableType,
                        updateCommand.Target.VariableName);
                    // Create the property to which will assign the correct value
                    var tenantProperty = DbExpressionBuilder.Property(variableReference, column);
                    // Create the tenantId where predicate, object representation of sql where tenantId = value statement
                    var tenantIdWherePredicate = DbExpressionBuilder.Equal(tenantProperty, DbExpression.FromString(userId));

                    // Remove potential assignment of tenantId for extra safety
                    var filteredSetClauses =
                        updateCommand.SetClauses.Cast<DbSetClause>()
                            .Where(sc => ((DbPropertyExpression)sc.Property).Property.Name != column);

                    // Construct the final clauses, object representation of sql insert command values
                    var finalSetClauses =
                        new ReadOnlyCollection<DbModificationClause>(new List<DbModificationClause>(filteredSetClauses));

                    // The initial predicate is the sql where statement
                    var initialPredicate = updateCommand.Predicate;
                    // Add to the initial statement the tenantId statement which translates in sql AND TenantId = 'value'
                    var finalPredicate = initialPredicate.And(tenantIdWherePredicate);

                    var newUpdateCommand = new DbUpdateCommandTree(
                        updateCommand.MetadataWorkspace,
                        updateCommand.DataSpace,
                        updateCommand.Target,
                        finalPredicate,
                        finalSetClauses,
                        updateCommand.Returning);

                    interceptionContext.Result = newUpdateCommand;
                    return;
                }
            }
        }
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

The first part of the code is exactly the same as the interception in insert command. The only difference here is that we don't create another set clause but a predicate which is the object representation of the SQL And statement. We explicitly do a logical AND with the original predicate and assign the final predicate in the new update command we construct.

***

#### Modification of delete command

The last piece to finish the puzzle is to modify the delete command before travelling to the database. The goal is exactly the same with update command. We want to append an extra where SQL statement in all delete commands. The code is also the same as in update command.

{% tabbed_codeblock Message POCO%}
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
            var userId = userIdclaim.Value;

            var deleteCommand = interceptionContext.Result as DbDeleteCommandTree;
            if (deleteCommand != null){
                var column = TenantAwareAttribute.GetTenantColumnName(deleteCommand.Target.VariableType.EdmType);
                if (!string.IsNullOrEmpty(column)){
                    // Create the variable reference in order to create the property
                    var variableReference = DbExpressionBuilder.Variable(deleteCommand.Target.VariableType,
                        deleteCommand.Target.VariableName);
                    // Create the property to which will assign the correct value
                    var tenantProperty = DbExpressionBuilder.Property(variableReference, column);
                    var tenantIdWherePredicate = DbExpressionBuilder.Equal(tenantProperty, DbExpression.FromString(userId));

                    // The initial predicate is the sql where statement
                    var initialPredicate = deleteCommand.Predicate;
                    // Add to the initial statement the tenantId statement which translates in sql AND TenantId = 'value'
                    var finalPredicate = initialPredicate.And(tenantIdWherePredicate);

                    var newDeleteCommand = new DbDeleteCommandTree(
                        deleteCommand.MetadataWorkspace,
                        deleteCommand.DataSpace,
                        deleteCommand.Target,
                        finalPredicate);

                    interceptionContext.Result = newDeleteCommand;
                    return ;
                }
            }
        }
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

We create manually again a predicate and do a logical AND with the original predicate of the command. After this we creating a new delete command and assign it as the interception result. There is no need to create different interceptors class per command of course. We can combine all of them in just one class as you can see in [this example][interceptor].

***

#### Final thoughts

This was the last part of a series of posts. You can find the first part [here][part1] and the second [here][part2]. I hope that these posts will become useful to other developers as this is the way we solved the multi tenancy data access problem in the project I am currently working on. Any comments or thoughts to improve the code are more than welcome. As I mentioned in the previous posts I have a public [Github repository][github] that contains a full project with the demonstrated code.

### [Part 1][part1]

### [Part 2][part2]

[part1]: /2014/11/17/Create-a-multitenant-application-with-Entity-Framework-Code-First-Part-1/
[part2]: /2014/11/18/Create-a-multitenant-application-with-Entity-Framework-Code-First-Part-2/
[miller]: http://romiller.com/
[session]: http://channel9.msdn.com/Events/TechEd/NorthAmerica/2014/DEV-B417#fbid=
[interceptor]: https://github.com/xabikos/EfMultitenant/blob/master/Multitenant.Interception/Infrastructure/TenantCommandTreeInterceptor.cs
[github]: https://github.com/xabikos/EfMultitenant/