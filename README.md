Решение задания Nº8: НТТР-сервер на Go
Вам нужно реализовать НТТР-сервер с использованием стандартной библиотеки net/http, показанный в лекции. Ниже приведён типовой пример такого сервера
с поддержкой REST API (CRUD для списка
пользователей).
Код сервера (main.go):

package main

import (
	"encoding/json"
	"log"
	"net/http"
	"strconv"
	"strings"
	"sync"
)

// User — модель данных
type User struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
	Age  int    `json:"age"`
}

// хранилище пользователей (in-memory)
var (
	users   = make(map[int]User)
	nextID  = 1
	mu      sync.RWMutex
)

func main() {
	// маршруты
	http.HandleFunc("/", homeHandler)
	http.HandleFunc("/users", usersHandler)
	http.HandleFunc("/users/", userByIDHandler)

	log.Println("Сервер запущен на :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

// homeHandler — приветствие
func homeHandler(w http.ResponseWriter, r *http.Request) {
	if r.URL.Path != "/" {
		http.NotFound(w, r)
		return
	}
	w.Header().Set("Content-Type", "text/plain")
	w.Write([]byte("Добро пожаловать в HTTP-сервер!\nДоступные эндпоинты:\n- GET /users\n- POST /users\n- GET /users/{id}\n- PUT /users/{id}\n- DELETE /users/{id}"))
}

// usersHandler — обработка списка пользователей (GET все, POST создать)
func usersHandler(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case http.MethodGet:
		getAllUsers(w, r)
	case http.MethodPost:
		createUser(w, r)
	default:
		http.Error(w, "Метод не разрешён", http.StatusMethodNotAllowed)
	}
}

// userByIDHandler — обработка одного пользователя (GET, PUT, DELETE)
func userByIDHandler(w http.ResponseWriter, r *http.Request) {
	// извлекаем ID из пути /users/{id}
	pathParts := strings.Split(r.URL.Path, "/")
	if len(pathParts) != 3 || pathParts[2] == "" {
		http.Error(w, "Неверный формат ID", http.StatusBadRequest)
		return
	}
	id, err := strconv.Atoi(pathParts[2])
	if err != nil {
		http.Error(w, "ID должен быть числом", http.StatusBadRequest)
		return
	}

	switch r.Method {
	case http.MethodGet:
		getUserByID(w, r, id)
	case http.MethodPut:
		updateUser(w, r, id)
	case http.MethodDelete:
		deleteUser(w, r, id)
	default:
		http.Error(w, "Метод не разрешён", http.StatusMethodNotAllowed)
	}
}

// getAllUsers — GET /users
func getAllUsers(w http.ResponseWriter, r *http.Request) {
	mu.RLock()
	defer mu.RUnlock()

	userList := make([]User, 0, len(users))
	for _, u := range users {
		userList = append(userList, u)
	}
	sendJSON(w, http.StatusOK, userList)
}

// createUser — POST /users
func createUser(w http.ResponseWriter, r *http.Request) {
	var u User
	if err := json.NewDecoder(r.Body).Decode(&u); err != nil {
		http.Error(w, "Ошибка декодирования JSON", http.StatusBadRequest)
		return
	}
	if u.Name == "" || u.Age <= 0 {
		http.Error(w, "Некорректные данные (name и age обязательны)", http.StatusBadRequest)
		return
	}

	mu.Lock()
	defer mu.Unlock()
	u.ID = nextID
	nextID++
	users[u.ID] = u
	sendJSON(w, http.StatusCreated, u)
}

// getUserByID — GET /users/{id}
func getUserByID(w http.ResponseWriter, r *http.Request, id int) {
	mu.RLock()
	defer mu.RUnlock()

	u, ok := users[id]
	if !ok {
		http.Error(w, "Пользователь не найден", http.StatusNotFound)
		return
	}
	sendJSON(w, http.StatusOK, u)
}

// updateUser — PUT /users/{id}
func updateUser(w http.ResponseWriter, r *http.Request, id int) {
	var updated User
	if err := json.NewDecoder(r.Body).Decode(&updated); err != nil {
		http.Error(w, "Ошибка декодирования JSON", http.StatusBadRequest)
		return
	}
	if updated.Name == "" || updated.Age <= 0 {
		http.Error(w, "Некорректные данные (name и age обязательны)", http.StatusBadRequest)
		return
	}

	mu.Lock()
	defer mu.Unlock()
	if _, ok := users[id]; !ok {
		http.Error(w, "Пользователь не найден", http.StatusNotFound)
		return
	}
	updated.ID = id
	users[id] = updated
	sendJSON(w, http.StatusOK, updated)
}

// deleteUser — DELETE /users/{id}
func deleteUser(w http.ResponseWriter, r *http.Request, id int) {
	mu.Lock()
	defer mu.Unlock()
	if _, ok := users[id]; !ok {
		http.Error(w, "Пользователь не найден", http.StatusNotFound)
		return
	}
	delete(users, id)
	w.WriteHeader(http.StatusNoContent)
}

// sendJSON — вспомогательная функция для отправки JSON
func sendJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}
