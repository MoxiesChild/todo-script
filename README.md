# todo-script
#!/bin/bash

# File to store todo tasks
TODO_FILE="todo_tasks.json"

# Initialize the JSON file if it doesn't exist
if [ ! -f $TODO_FILE ]; then
  echo "[]" > $TODO_FILE
fi

# Function to generate a unique identifier
generate_id() {
  echo $(uuidgen)
}

# Function to validate date and time
validate_datetime() {
  date -d "$1" "+%Y-%m-%d %H:%M" > /dev/null 2>&1
}

# Function to create a task
create_task() {
  echo "Enter title (required): "
  read title
  if [ -z "$title" ]; then
    echo "Error: Title is required" >&2
    exit 1
  fi

  echo "Enter description: "
  read description

  echo "Enter location: "
  read location

  echo "Enter due date and time (YYYY-MM-DD HH:MM, required): "
  read due_date
  if ! validate_datetime "$due_date"; then
    echo "Error: Invalid date and time format" >&2
    exit 1
  fi

  id=$(generate_id)
  task=$(jq -n --arg id "$id" --arg title "$title" --arg description "$description" --arg location "$location" --arg due_date "$due_date" --argjson completed false '{
    id: $id,
    title: $title,
    description: $description,
    location: $location,
    due_date: $due_date,
    completed: $completed
  }')

  jq ". += [$task]" $TODO_FILE > tmp.$$.json && mv tmp.$$.json $TODO_FILE
  echo "Task created with ID: $id"
}

# Function to update a task
update_task() {
  echo "Enter task ID to update: "
  read id

  task=$(jq -r --arg id "$id" '.[] | select(.id == $id)' $TODO_FILE)
  if [ -z "$task" ]; then
    echo "Error: Task not found" >&2
    exit 1
  fi

  echo "Enter new title (leave blank to keep current): "
  read title
  [ -z "$title" ] && title=$(echo $task | jq -r '.title')

  echo "Enter new description (leave blank to keep current): "
  read description
  [ -z "$description" ] && description=$(echo $task | jq -r '.description')

  echo "Enter new location (leave blank to keep current): "
  read location
  [ -z "$location" ] && location=$(echo $task | jq -r '.location')

  echo "Enter new due date and time (YYYY-MM-DD HH:MM, leave blank to keep current): "
  read due_date
  if [ -n "$due_date" ] && ! validate_datetime "$due_date"; then
    echo "Error: Invalid date and time format" >&2
    exit 1
  fi
  [ -z "$due_date" ] && due_date=$(echo $task | jq -r '.due_date')

  echo "Is the task completed? (yes/no, leave blank to keep current): "
  read completed
  case $completed in
    yes) completed=true ;;
    no) completed=false ;;
    *) completed=$(echo $task | jq -r '.completed') ;;
  esac

  updated_task=$(jq -n --arg id "$id" --arg title "$title" --arg description "$description" --arg location "$location" --arg due_date "$due_date" --argjson completed $completed '{
    id: $id,
    title: $title,
    description: $description,
    location: $location,
    due_date: $due_date,
    completed: $completed
  }')

  jq --arg id "$id" 'map(if .id == $id then . = $ARGS.named else . end)' --argjson named "$updated_task" $TODO_FILE > tmp.$$.json && mv tmp.$$.json $TODO_FILE
  echo "Task updated"
}

# Function to delete a task
delete_task() {
  echo "Enter task ID to delete: "
  read id

  jq --arg id "$id" 'del(.[] | select(.id == $id))' $TODO_FILE > tmp.$$.json && mv tmp.$$.json $TODO_FILE
  echo "Task deleted"
}

# Function to show a task
show_task() {
  echo "Enter task ID to show: "
  read id

  task=$(jq --arg id "$id" '.[] | select(.id == $id)' $TODO_FILE)
  if [ -z "$task" ]; then
    echo "Error: Task not found" >&2
    exit 1
  fi

  echo $task | jq .
}

# Function to list tasks of a given day
list_tasks() {
  echo "Enter date (YYYY-MM-DD): "
  read date

  tasks=$(jq --arg date "$date" '[.[] | select(.due_date | startswith($date))]' $TODO_FILE)

  echo "Completed tasks:"
  echo $tasks | jq '.[] | select(.completed == true)'

  echo "Uncompleted tasks:"
  echo $tasks | jq '.[] | select(.completed == false)'
}

# Function to search for a task by title
search_task() {
  echo "Enter title to search: "
  read title

  tasks=$(jq --arg title "$title" '[.[] | select(.title | test($title; "i"))]' $TODO_FILE)
  echo $tasks | jq .
}

# Function to display today's tasks
today_tasks() {
  date=$(date "+%Y-%m-%d")
  tasks=$(jq --arg date "$date" '[.[] | select(.due_date | startswith($date))]' $TODO_FILE)

  echo "Completed tasks:"
  echo $tasks | jq '.[] | select(.completed == true)'

  echo "Uncompleted tasks:"
  echo $tasks | jq '.[] | select(.completed == false)'
}

# Main script logic
case $1 in
  create)
    create_task
    ;;
  update)
    update_task
    ;;
  delete)
    delete_task
    ;;
  show)
    show_task
    ;;
  list)
    list_tasks
    ;;
  search)
    search_task
    ;;
  *)
    today_tasks
    ;;
esac
