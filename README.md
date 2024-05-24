# check-my-code

const express = require('express')
const {open} = require('sqlite')
const sqlite3 = require('sqlite3')
const path = require('path')
var  isValid = require('date-fns/isValid')
const format = require('date-fns/format')
const isMatch = require('date-fns/isMatch')
const app = express()
app.use(express.json());

const dbPath = path.join(__dirname, 'todoApplication.db');

let db = null;

const initializeDbAndServer = async () => {
  try {
    db = await open({
      filename: dbPath,
      driver: sqlite3.Database,
    })
    app.listen(3000, () =>
      console.log('Server Running at http://localhost:3000/'),
    )
  } catch (error) {
    console.log(`DB Error: ${error.message}`)
    process.exit(1)
  }
}

initializeDbAndServer()

const hasPriorityAndStatusProperties = (requestQuery) => {
  return (
    requestQuery.priority !== undefined && requestQuery.status !== undefined
  );
};

const hasCategoryAndStatusProperties = (requestQuery) => {
  return requestQuery.category !== undefined && requestQuery.status !== undefined
};

const hasCategoryAndPriorityProperties = (requestQuery) => {
  return (
    requestQuery.category !== undefined && requestQuery.priority !== undefined
  );
};

const hasPriorityProperty = (requestQuery) => {
  return requestQuery.priority !== undefined
};

const hasStatusProperty = (requestQuery) => {
  return requestQuery.status !== undefined
};

const hasCategoryProperty = (requestQuery) => {
  return requestQuery.category !== undefined;
};

const hasSearchProperty = (requestQuery) => {
  return requestQuery.search_q !== undefined;
};

const convertDataIntoResponseObject = (dbObject) => {
  return {
    id: dbObject.id,
    todo: dbObject.todo,
    priority: dbObject.priority,
    status: dbObject.status,
    category: dbObject.category,
    dueDate: dbObject.due_date,
  };
};

app.get('/todos/', async (request, response) => {
    const { search_q='', priority, status, category}=request.query;

    let data=null;

    let getTodosQuery='';

    switch (true) {
        case hasStatusProperty(request.query):
        if (status==="TO DO" || status === "IN PROGRESS" || status==="DONE") {
            getTodosQuery=`
                 SELECT * FROM todo WHERE status ='${status}';`;
            data=await db.all(getTodosQuery);
            response.send(data.map((eachItem)=>convertDataIntoResponseObject(eachItem))
            );
        } else{
            response.status(400);
            response.send('Invalid Todo Status');
        }
        break;

         
         case hasPriorityProperty(request.query):
         if (priority==="HIGH" || priority === "MEDIUM" || priority==="LOW"){
            getTodosQuery=`
                 SELECT * FROM todo WHERE priority = '${priority}';`;

            data = await db.all(getTodosQuery);
            response.send(data.map((eachItem)=>convertDataIntoResponseObject(eachItem))
            );
         } else {
            response.status(400);
            response.send('Invalid Todo Priority');

         }
          break;
    case hasPriorityAndStatusProperties(request.query):
    if (priority==="HIGH" || priority === "MEDIUM" || priority==="LOW"){
        if (status==="TO DO" || status === "IN PROGRESS" || status==="DONE"){
            getTodosQuery=`
                  SELECT * FROM todo
                  WHERE  priority= '${priority}'
                  AND status ='${status}';`;
            data=await db.all(getTodosQuery);
            response.send(data.map((eachItem)=>convertDataIntoResponseObject(eachItem))
            );
        } else {
            response.status(400);
            response.send('Invalid Todo Status');
        }
    } else{
        response.status(400);
        response.send('Invalid Todo Priority');

    }
       break;
    case hasSearchProperty(request.query):
     getTodosQuery=`SELECT * FROM todo WHERE todo LIKE '%${search_q}%';`;

     data=await db.all(getTodosQuery);
     response.send(data.map((eachItem)=>convertDataIntoResponseObject(eachItem))
     );
     break;

    
    case hasCategoryAndStatusProperties(request.query):
     if (category==="WORK" || category === "HOME" || category==="LEARNING"){
        if (status==="TO DO" || status === "IN PROGRESS" ||status==="DONE"){
            getTodosQuery=`
              SELECT * FROM todo
              WHERE category ='${category}'
               AND  status   ='${status}';`;


            data=await db.all(getTodosQuery);
            response.send(data.map((eachItem)=>convertDataIntoResponseObject(eachItem))
            );

        } else {
            response.status(400);
            response.send('Invalid Todo Status');

        }
        

     }else{
        response.status(400);
        response.send('Invalid Todo Category');

     }
     break;

    case hasCategoryProperty(request.query):
    if (category==="WORK" ||category === "HOME" || category==="LEARNING") {
        getTodosQuery=`
              SELECT * FROM todo
              WHERE category ='${category}';`;
        data= await db.all(getTodosQuery);
            response.send(data.map((eachItem) => convertDataIntoResponseObject(eachItem))
            );
        } else{
            response.status(400);
            response.send('Invalid Todo Category');

        }
        break;
    case hasCategoryAndPriorityProperties(request.query):
        if (category==="WORK" ||category === "HOME" || category==="LEARNING"){
        if (priority==="HIGH" || priority === "MEDIUM" || priority==="LOW"){
            getTodosQuery=`
              SELECT * FROM todo
              WHERE category ='${category}'
               AND  priority   ='${priority}';`;


            data= await db.all(getTodosQuery);
            response.send(data.map((eachItem)=>convertDataIntoResponseObject(eachItem))
            );

        } else {
            response.status(400);
            response.send('Invalid Todo Priority')

        }
        

     }else{
        response.status(400);
        response.send('Invalid Todo Category')

     }
     break;
    
    default:
        getTodosQuery=`SELECT * FROM todo;`;
        data=await db.all(getTodosQuery);
            response.send(data.map((eachItem)=>convertDataIntoResponseObject(eachItem))
            );
    }

});

app.get('/todos/:todoId/', async (request, response)=>{
    const {todoId}=request.params;
    const getTodosQuery=`SELECT * FROM todo
             WHERE id=${todoId};`;
    const dbResponse= await db.get(getTodosQuery);
    response.send(convertDataIntoResponseObject(dbResponse));
});

app.get('/agenda/', async (request, response)=>{
    const {date}=request.query;

    if (isMatch(date, 'yyyy-MM-dd')) {
        const newDate= format(new Date (date), 'yyyy-MM-dd');

        const getDateQuery=`SELECT * FROM todo WHERE due_date='${newDate}';`;
  
        const dbResponse=await db.all(getDateQuery)

        response.send(dbResponse.map((eachItem)=>convertDataIntoResponseObject(eachItem))
        );
        
    } else {
        response.status(400);
        response.send('Invalid Due Date')


    }
});



app.post('/todos/', async (request, response)=>{
    const {id, todo, priority, status, dueDate}=request.body;

    if (priority==="HIGH" || priority === "MEDIUM" || priority==="LOW"){
        if (status==="TO DO" || status === "IN PROGRESS" ||status==="DONE"){
            if (category==="WORK" ||category === "HOME" || category==="LEARNING"){
                if (isMatch(date, 'YYYY-MM-dd')){
                    const newDueDate= format(new Date(date), 'YYYY-MM-dd');

                    const postQuery=`
                          INSERT INTO
                          todo(id, todo, priority, status, category, due_date)
                          VALUES
                          (${id}, '${todo}', '${priority}', '${status}', '${category}', '${newDueDate}');`;
                    await db.run(postQuery)
                    response.send("Todo Successfully Added")

                } else{
                    response.status(400);
                    response.send('Invalid Due Date')

                }
            } else {
                response.status(400);
                response.send('Invalid Todo Category')

            }

        } else{
            response.status(400);
            response.send('Invalid Todo Status')
        }
    } else {
        response.status(400);
        response.send('Invalid Todo Priority')

    }
});


app.put('/todos/:todoId/', async (request, response)=>{
    const {todoId} = request.params;

    let updateColumn='';
    const requestBody=request.body;

    const previousTodoQuery= `SELECT * FROM todo WHERE id = ${todoId};`;
    const previousTodo = await db.get(previousTodoQuery);

    const {
    todo= previousTodo.todo,
    priority= previousTodo.priority,
    status= previousTodo.status,
    category= previousTodo.category,
    dueDate= previousTodo.due_date,

    }= request.body;

  let updateTodo;

  switch (true) {
    case requestBody.status!==undefined:
      if (status==="TO DO" || status === "IN PROGRESS" ||status==="DONE"){
        updateTodo=`
              UPDATE
              TODO
              SET
                 todo='${todo}',
                 priority='${priority}',
                  status='${status}',
                  category='${category}', 
                  due_date='${dueDate}'
                  WHERE id = ${todoId};`;
                  
        await db.run(updateTodo);
        response.send('Status Updated')
      } else{
        response.status(400);
        response.send('Invalid Todo Status')

      };
      break;

    case requestBody.priority!==undefined:
    if (priority==="HIGH" || priority === "MEDIUM" || priority==="LOW") {
        updateTodo=`
              UPDATE
              TODO
              SET
                 todo='${todo}',
                 priority='${priority}',
                  status='${status}',
                  category='${category}', 
                  due_date='${dueDate}'
                  WHERE id = ${todoId};`;
                  
        await db.run(updateTodo);
        response.send('Priority Updated')


    } else{
        response.status(400);
        response.send('Invalid Todo Priority');

    };
    break;
    case requestBody.todo!==undefined:
    updateTodo=`
              UPDATE
              TODO
              SET
                 todo='${todo}',
                 priority='${priority}',
                  status='${status}',
                  category='${category}', 
                  due_date='${dueDate}'
                  WHERE id = ${todoId};`;
                  
        await db.run(updateTodo);
        response.send('Todo Updated');

        break;
     
    case requestBody.category!==undefined;
    if (category==="WORK" ||category === "HOME" || category==="LEARNING"){
        updateTodo=`
              UPDATE
              TODO
              SET
                 todo='${todo}',
                 priority='${priority}',
                  status='${status}',
                  category='${category}', 
                  due_date='${dueDate}'
                  WHERE id = ${todoId};`;
                  
        await db.run(updateTodo);
        response.send('Category Updated')
    }else{
        response.status(400);
        response.send('Invalid Todo Category')
    }

    break;

    case requestBody.dueDate!==undefined:
       
       if (isMatch(date, 'YYYY-MM-dd')) {
        const newDate= format(new Date (date), 'YYYY-MM-dd');

            updateTodo=`
              UPDATE
              TODO
              SET
                 todo='${todo}',
                 priority='${priority}',
                  status='${status}',
                  category='${category}', 
                  due_date='${dueDate}'
                  WHERE id = ${todoId};`;
                  
        await db.run(updateTodo);
        response.send('Due Date Updated') 

       }else{
        response.status(400);
        response.send('Invalid Due Date')

       }

       break;
    }
});

app.delete('/todos/:todoId/', async (request, response)=>{
    const {todoId}=request.parmas;
    const deleteTodoQuery=`
        DELETE
        FROM
            todo
        WHERE
            id=${todoId};`;
    await db.run(deleteTodoQuery)
    response.send('Todo Deleted')
});



module.exports = app;
