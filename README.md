We will be deploying a GraphQL API that uses embeddings, vector similarity search, and relational data to return a set of products that could be used by a chat application leveraging a Large Language Model (LLM). You will create a stored procedure that will be used by the GraphQL API for taking in questions and returning products.

Let's create a new stored procedure to create a new flow that not only uses vector similarity search to get products based on a question asked by a user, but to take the results, pass them to Azure OpenAI Chat Completion, and craft an answer they would typically see with an AI chat application.

The first step in augmenting our RAG application API is to create a stored procedure that takes the retrieved products and passes them in a prompt to an Azure OpenAI Chat Completion REST endpoint. The prompt consists of telling the endpoint who they are, what products they have to work with, and the exact question that was asked by the user.

CREATE OR ALTER PROCEDURE [dbo].[prompt_answer]
@user_question nvarchar(max),
@products nvarchar(max),
@answer nvarchar(max) output

AS

declare @url nvarchar(4000) = N'https://AI_ENDPOINT_SERVERNAME.openai.azure.com/openai/deployments/gpt-4/chat/completions?api-version=2024-06-01';
declare @payload nvarchar(max) = N'{
    "messages": [
        {
            "role": "system",
            "content": "You are a sales assistant who helps customers find the right products for their question and activities."
        },
        {
            "role": "user",
            "content": "The products available are the following: ' + @products + '"
        },
        {
            "role": "user",
            "content": " ' + @user_question + '"
        }
    ]
}';

declare @ret int, @response nvarchar(max);

exec @ret = sp_invoke_external_rest_endpoint
    @url = @url,
    @method = 'POST', 
    @payload = @payload,
    @credential = [https://AI_ENDPOINT_SERVERNAME.openai.azure.com/],    
    @timeout = 230,
    @response = @response output;

select json_value(@response, '$.result.choices[0].message.content');

GO

Now that you have created the chat completion stored procedure, we need to create a new find_products_chat stored procedure that adds a call to this chat completion endpoint.

create or alter procedure [dbo].[find_products_chat]
@text nvarchar(max),
@top int = 3,
@min_similarity decimal(19,16) = 0.70
as
if (@text is null) return;
declare @retval int, @qv vector(1536), @products_json nvarchar(max), @answer nvarchar(max);
exec @retval = dbo.create_embeddings @text, @qv output;
if (@retval != 0) return;
with vector_results as (
SELECT 
        p.Name as product_name,
        ISNULL(p.Color,'No Color') as product_color,
        c.Name as category_name,
        m.Name as model_name,
        d.Description as product_description,
        p.ListPrice as list_price,
        p.weight as product_weight,
        vector_distance('cosine', @qv, p.embeddings) AS distance
FROM
    [SalesLT].[Product] p,
    [SalesLT].[ProductCategory] c,
    [SalesLT].[ProductModel] m,
    [SalesLT].[vProductAndDescription] d
where p.ProductID = d.ProductID
and p.ProductCategoryID = c.ProductCategoryID
and p.ProductModelID = m.ProductModelID
and p.ProductID = d.ProductID
and d.Culture = 'en')
select
top(@top)
@products_json = (STRING_AGG (CONVERT(NVARCHAR(max),CONCAT( 
                                product_name, ' ' ,
                                product_color, ' ',
                                category_name, ' ', 
                                model_name, ' ', 
                                product_description, ' ',
                                list_price, ' ',
                                product_weight )), CHAR(13)))
from vector_results
where (1-distance) > @min_similarity
group by distance
order by    
    distance asc;

set @products_json = (select REPLACE(REPLACE(@products_json, CHAR(13), ' , '), CHAR(10), ' , '));

exec [dbo].[prompt_answer] @text, @products_json, @answer output;

GO


The last step before we can create a GraphQL endpoint is to wrap the new find products chat stored procedure. Copy/Paste the below T-SQL Code in a new query window and Run the code:

create or alter procedure [find_products_chat_api]
@text nvarchar(max)
as 
exec find_products_chat @text
with RESULT SETS
(    
    (    
        answer NVARCHAR(max)
    )
)
GO

ou can test this new procedure to see how Azure OpenAI will answer a question with product data. Copy/Paste the below T-SQL Code in a new query window and Run the code:

exec find_products_chat_api 'I am looking for a red bike'

<img width="1211" height="520" alt="image" src="https://github.com/user-attachments/assets/72dbe704-80ef-4c5b-aa10-47f36793ec79" />

2. Create GraphQL API

You will now be on the GraphQL Query editor page. Copy/Paste the below code in the GraphQL query editor.

query {
executefind_products_api(text: "I am looking for a red bike") {
        product_name
        product_color
        category_name
        model_name
        product_description
        list_price
        product_weight
        distance 
    }
}

The API you just created could now be handed off to an application developer to be included in a RAG application that uses vector similarity search and data from the database.

<img width="1300" height="745" alt="image" src="https://github.com/user-attachments/assets/fe61acc5-b3ae-479d-8cad-727e77d30daf" />







