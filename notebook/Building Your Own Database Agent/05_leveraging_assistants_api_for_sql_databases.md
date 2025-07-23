# Lesson 5: Leveraging Assistants API for SQL Databases

## Setup


```python
from openai import AzureOpenAI
import json
import os
```

## Import the helper function

To access the ``Helper.py`` file, please go to the ``File`` menu and select ``Open...``.


```python
import Helper
from Helper import get_positive_cases_for_state_on_date
from Helper import get_hospitalized_increase_for_state_on_date
```

## Launch the Assistant API

**Note**: The pre-configured cloud resource grants you access to the Azure OpenAI GPT model. The key and endpoint provided below are intended for teaching purposes only. Your notebook environment is already set up with the necessary keys, which may differ from those used by the instructor during the filming.


```python
client = AzureOpenAI(
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    api_version="2024-02-15-preview",
    azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
    )

# I) Create assistant
assistant = client.beta.assistants.create(
  instructions="""You are an assistant answering questions 
                  about a Covid dataset.""",
  model="gpt-4-1106", 
  tools=Helper.tools_sql)

# II) Create thread
thread = client.beta.threads.create()
print(thread)
```

    Thread(id='thread_5yAdw4ZNVEeB6WIlkTaG8e9N', created_at=1753234313, metadata={}, object='thread')



```python
# III) Add message
message = client.beta.threads.messages.create(
    thread_id=thread.id,
    role="user",
    content="""how many hospitalized people we had in Alaska
               the 2021-03-05?"""
)
print(message)
```

    ThreadMessage(id='msg_4CtN1y11leqG10OfjV1JqHxB', assistant_id=None, content=[MessageContentText(text=Text(annotations=[], value='how many hospitalized people we had in Alaska\n               the 2021-03-05?'), type='text')], created_at=1753234364, file_ids=[], metadata={}, object='thread.message', role='user', run_id=None, thread_id='thread_5yAdw4ZNVEeB6WIlkTaG8e9N')



```python
# example output:
'''
ThreadMessage(
    id="msg_4CtN1y11leqG10OfjV1JqHxB",
    assistant_id=None,
    content=[
        MessageContentText(
            text=Text(
                annotations=[],
                value="how many hospitalized people we had in Alaska\n the 2021-03-05?",
            ),
            type="text",
        )
    ],
    created_at=1753234364,
    file_ids=[],
    metadata={},
    object="thread.message",
    role="user",
    run_id=None,
    thread_id="thread_5yAdw4ZNVEeB6WIlkTaG8e9N",
)
'''
```


```python
messages = client.beta.threads.messages.list(
  thread_id=thread.id
)

print(messages.model_dump_json(indent=2))
```

    {
      "data": [
        {
          "id": "msg_4CtN1y11leqG10OfjV1JqHxB",
          "assistant_id": null,
          "content": [
            {
              "text": {
                "annotations": [],
                "value": "how many hospitalized people we had in Alaska\n               the 2021-03-05?"
              },
              "type": "text"
            }
          ],
          "created_at": 1753234364,
          "file_ids": [],
          "metadata": {},
          "object": "thread.message",
          "role": "user",
          "run_id": null,
          "thread_id": "thread_5yAdw4ZNVEeB6WIlkTaG8e9N"
        }
      ],
      "object": "list",
      "first_id": "msg_4CtN1y11leqG10OfjV1JqHxB",
      "last_id": "msg_4CtN1y11leqG10OfjV1JqHxB",
      "has_more": false
    }



```python
# IV) Run assistant on thread

run = client.beta.threads.runs.create(
  thread_id=thread.id,
  assistant_id=assistant.id,
)
```

## Leverage the function calling with Assistants API


```python
import time
from IPython.display import clear_output

start_time = time.time()

status = run.status

while status not in ["completed", "cancelled", "expired", "failed"]:
    time.sleep(5)
    run = client.beta.threads.runs.retrieve(
        thread_id=thread.id,run_id=run.id
    )
    print("Elapsed time: {} minutes {} seconds".format(
        int((time.time() - start_time) // 60),
        int((time.time() - start_time) % 60))
         )
    status = run.status
    print(f'Status: {status}')
    if (status=="requires_action"):
        available_functions = {
            "get_positive_cases_for_state_on_date": get_positive_cases_for_state_on_date,
            "get_hospitalized_increase_for_state_on_date":get_hospitalized_increase_for_state_on_date
        }

        tool_outputs = []
        for tool_call in run.required_action.submit_tool_outputs.tool_calls:
            function_name = tool_call.function.name
            function_to_call = available_functions[function_name]
            function_args = json.loads(tool_call.function.arguments)
            function_response = function_to_call(
                state_abbr=function_args.get("state_abbr"),
                specific_date=function_args.get("specific_date"),
            )
            print(function_response)
            print(tool_call.id)
            tool_outputs.append(
                { "tool_call_id": tool_call.id,
                 "output": str(function_response)
                }
            )

        run = client.beta.threads.runs.submit_tool_outputs(
          thread_id=thread.id,
          run_id=run.id,
          tool_outputs = tool_outputs
        )


messages = client.beta.threads.messages.list(
  thread_id=thread.id
)

print(messages)
```

    Elapsed time: 0 minutes 5 seconds
    Status: requires_action
    {'date': '2021-03-05', 'hospitalizedIncrease': 3}
    call_gHDYvXIpQerNSAS67usMwiiU
    Elapsed time: 0 minutes 10 seconds
    Status: completed
    SyncCursorPage[ThreadMessage](data=[ThreadMessage(id='msg_eT67RVBh3QtXnJlbZTy6dyNA', assistant_id='asst_uWN42Xj7CnnPgsSZ1HdF0OJv', content=[MessageContentText(text=Text(annotations=[], value='On March 5, 2021, Alaska reported a daily increase of 3 hospitalized people due to Covid.'), type='text')], created_at=1753234620, file_ids=[], metadata={}, object='thread.message', role='assistant', run_id='run_JjogxMpvjFF7qBlDpP7Jb4PN', thread_id='thread_5yAdw4ZNVEeB6WIlkTaG8e9N'), ThreadMessage(id='msg_4CtN1y11leqG10OfjV1JqHxB', assistant_id=None, content=[MessageContentText(text=Text(annotations=[], value='how many hospitalized people we had in Alaska\n               the 2021-03-05?'), type='text')], created_at=1753234364, file_ids=[], metadata={}, object='thread.message', role='user', run_id=None, thread_id='thread_5yAdw4ZNVEeB6WIlkTaG8e9N')], object='list', first_id='msg_eT67RVBh3QtXnJlbZTy6dyNA', last_id='msg_4CtN1y11leqG10OfjV1JqHxB', has_more=False)



```python
print(messages.model_dump_json(indent=2))
```

    {
      "data": [
        {
          "id": "msg_eT67RVBh3QtXnJlbZTy6dyNA",
          "assistant_id": "asst_uWN42Xj7CnnPgsSZ1HdF0OJv",
          "content": [
            {
              "text": {
                "annotations": [],
                "value": "On March 5, 2021, Alaska reported a daily increase of 3 hospitalized people due to Covid."
              },
              "type": "text"
            }
          ],
          "created_at": 1753234620,
          "file_ids": [],
          "metadata": {},
          "object": "thread.message",
          "role": "assistant",
          "run_id": "run_JjogxMpvjFF7qBlDpP7Jb4PN",
          "thread_id": "thread_5yAdw4ZNVEeB6WIlkTaG8e9N"
        },
        {
          "id": "msg_4CtN1y11leqG10OfjV1JqHxB",
          "assistant_id": null,
          "content": [
            {
              "text": {
                "annotations": [],
                "value": "how many hospitalized people we had in Alaska\n               the 2021-03-05?"
              },
              "type": "text"
            }
          ],
          "created_at": 1753234364,
          "file_ids": [],
          "metadata": {},
          "object": "thread.message",
          "role": "user",
          "run_id": null,
          "thread_id": "thread_5yAdw4ZNVEeB6WIlkTaG8e9N"
        }
      ],
      "object": "list",
      "first_id": "msg_eT67RVBh3QtXnJlbZTy6dyNA",
      "last_id": "msg_4CtN1y11leqG10OfjV1JqHxB",
      "has_more": false
    }


## Add the code interpreter


```python
file = client.files.create(
  file=open("./data/all-states-history.csv", "rb"),
  purpose='assistants'
)
assistant = client.beta.assistants.create(
  instructions="You are an assitant answering questions about a Covid dataset.",
  model="gpt-4-1106", 
  tools=[{"type": "code_interpreter"}],
  file_ids=[file.id]
)
thread = client.beta.threads.create()
print(thread)
message = client.beta.threads.messages.create(
    thread_id=thread.id,
    role="user",
    content="how many hospitalized people we had in Alaska the 2021-03-05?"
)
print(message)
run = client.beta.threads.runs.create(
  thread_id=thread.id,
  assistant_id=assistant.id,
)
```

    Thread(id='thread_Yh7drsJdxV5sw5QLCRFnD9D3', created_at=1753234832, metadata={}, object='thread')
    ThreadMessage(id='msg_sRVwJaz3ZwoytQicEBThVd18', assistant_id=None, content=[MessageContentText(text=Text(annotations=[], value='how many hospitalized people we had in Alaska the 2021-03-05?'), type='text')], created_at=1753234833, file_ids=[], metadata={}, object='thread.message', role='user', run_id=None, thread_id='thread_Yh7drsJdxV5sw5QLCRFnD9D3')



```python
status = run.status
start_time = time.time()
while status not in ["completed", "cancelled", "expired", "failed"]:
    time.sleep(5)
    run = client.beta.threads.runs.retrieve(
        thread_id=thread.id,
        run_id=run.id
    )
    print("Elapsed time: {} minutes {} seconds".format(
        int((time.time() - start_time) // 60),
        int((time.time() - start_time) % 60))
         )
    status = run.status
    print(f'Status: {status}')
    clear_output(wait=True)


messages = client.beta.threads.messages.list(
  thread_id=thread.id
)

print(messages.model_dump_json(indent=2))
```

    {
      "data": [
        {
          "id": "msg_CxRrHROu5hPOyaOsCSu56ow1",
          "assistant_id": "asst_Al60iXUqLEL0zhczucEk8LW9",
          "content": [
            {
              "text": {
                "annotations": [],
                "value": "On 2021-03-05, there were a total of **1293 hospitalized individuals** in Alaska (`AK`)."
              },
              "type": "text"
            }
          ],
          "created_at": 1753234840,
          "file_ids": [],
          "metadata": {},
          "object": "thread.message",
          "role": "assistant",
          "run_id": "run_P9pXztz1uRhf9CbnHnnQ5umP",
          "thread_id": "thread_Yh7drsJdxV5sw5QLCRFnD9D3"
        },
        {
          "id": "msg_PgtIPkIxkkkVb85frdDxxRDv",
          "assistant_id": "asst_Al60iXUqLEL0zhczucEk8LW9",
          "content": [
            {
              "text": {
                "annotations": [],
                "value": "The dataset contains various columns, including date, state, and hospitalization data, such as:\n\n- `hospitalized`: Total hospitalized individuals.\n- `hospitalizedCurrently`: Current hospitalized individuals.\n\nTo answer your question regarding the number of hospitalized people in Alaska (`state=\"AK\"`) on 2021-03-05, we will filter the data accordingly. Let me proceed."
              },
              "type": "text"
            }
          ],
          "created_at": 1753234837,
          "file_ids": [],
          "metadata": {},
          "object": "thread.message",
          "role": "assistant",
          "run_id": "run_P9pXztz1uRhf9CbnHnnQ5umP",
          "thread_id": "thread_Yh7drsJdxV5sw5QLCRFnD9D3"
        },
        {
          "id": "msg_Oycp3yjGtRHs5LETH3hk1P8I",
          "assistant_id": "asst_Al60iXUqLEL0zhczucEk8LW9",
          "content": [
            {
              "text": {
                "annotations": [],
                "value": "Let me first inspect the contents of the file you have uploaded to understand its structure and locate the relevant data. Then, I can identify the hospitalization information you are interested in."
              },
              "type": "text"
            }
          ],
          "created_at": 1753234833,
          "file_ids": [],
          "metadata": {},
          "object": "thread.message",
          "role": "assistant",
          "run_id": "run_P9pXztz1uRhf9CbnHnnQ5umP",
          "thread_id": "thread_Yh7drsJdxV5sw5QLCRFnD9D3"
        },
        {
          "id": "msg_sRVwJaz3ZwoytQicEBThVd18",
          "assistant_id": null,
          "content": [
            {
              "text": {
                "annotations": [],
                "value": "how many hospitalized people we had in Alaska the 2021-03-05?"
              },
              "type": "text"
            }
          ],
          "created_at": 1753234833,
          "file_ids": [],
          "metadata": {},
          "object": "thread.message",
          "role": "user",
          "run_id": null,
          "thread_id": "thread_Yh7drsJdxV5sw5QLCRFnD9D3"
        }
      ],
      "object": "list",
      "first_id": "msg_CxRrHROu5hPOyaOsCSu56ow1",
      "last_id": "msg_sRVwJaz3ZwoytQicEBThVd18",
      "has_more": false
    }

