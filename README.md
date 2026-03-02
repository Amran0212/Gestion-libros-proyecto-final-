# Gestion-libros-proyecto-final-
Codificación

// main.go - Punto de entrada de la aplicación web
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"log"
	"net/http"
	"strconv"
	"strings"
	"sync"
	"time"
)

// =======================
// ERRORES PERSONALIZADOS
// =======================
var (
	ErrLibroNoEncontrado    = errors.New("libro no encontrado")
	ErrIDInvalido           = errors.New("ID inválido: debe ser mayor que cero")
	ErrDatosIncompletos     = errors.New("datos incompletos: título y autor son obligatorios")
	ErrRepositorioNoListo   = errors.New("repositorio no inicializado correctamente")
	ErrUsuarioNoEncontrado  = errors.New("usuario no encontrado")
)

// =======================
// MODELO DE DATOS - LIBRO (Encapsulamiento POO)
// =======================
type Libro struct {
	ID        int    `json:"id"`
	Titulo    string `json:"titulo"`
	Autor     string `json:"autor"`
	Categoria string `json:"categoria,omitempty"`
	Formato   string `json:"formato,omitempty"`
	FechaRegistro time.Time `json:"fecha_registro"`
}

// NewLibro - Constructor con validación (principio de encapsulamiento)
func NewLibro(id int, titulo, autor, categoria, formato string) (*Libro, error) {
	titulo = strings.TrimSpace(titulo)
	autor = strings.TrimSpace(autor)

	if id < 0 {
		return nil, ErrIDInvalido
	}
	if titulo == "" || autor == "" {
		return nil, ErrDatosIncompletos
	}

	return &Libro{
		ID:            id,
		Titulo:        titulo,
		Autor:         autor,
		Categoria:     strings.TrimSpace(categoria),
		Formato:       strings.TrimSpace(formato),
		FechaRegistro: time.Now(),
	}, nil
}

// Getters - Acceso controlado a atributos privados (simulados)
func (l *Libro) GetID() int              { return l.ID }
func (l *Libro) GetTitulo() string       { return l.Titulo }
func (l *Libro) GetAutor() string        { return l.Autor }
func (l *Libro) GetCategoria() string    { return l.Categoria }
func (l *Libro) GetFormato() string      { return l.Formato }

// String - Implementación de interfaz fmt.Stringer para debugging
func (l *Libro) String() string {
	return fmt.Sprintf("ID: %d | Título: %s | Autor: %s | Categoría: %s",
		l.ID, l.Titulo, l.Autor, l.Categoria)
}

// =======================
// MODELO DE DATOS - USUARIO
// =======================
type Usuario struct {
	ID       int    `json:"id"`
	Nombre   string `json:"nombre"`
	Email    string `json:"email"`
	Rol      string `json:"rol"` // "admin" o "lector"
	Activo   bool   `json:"activo"`
}

func NewUsuario(id int, nombre, email, rol string) (*Usuario, error) {
	if nombre == "" || email == "" {
		return nil, errors.New("nombre y email son obligatorios")
	}
	return &Usuario{
		ID:     id,
		Nombre: nombre,
		Email:  email,
		Rol:    rol,
		Activo: true,
	}, nil
}

// =======================
// INTERFAZ DE REPOSITORIO (Abstracción - Principio de Inversión de Dependencias)
// =======================
type RepositorioLibros interface {
	Agregar(libro *Libro) error
	Listar() ([]*Libro, error)
	BuscarPorID(id int) (*Libro, error)
	Buscar(criterio string) ([]*Libro, error)
	Actualizar(id int, libro *Libro) error
	Eliminar(id int) error
	SiguienteID() int
}

type RepositorioUsuarios interface {
	Agregar(usuario *Usuario) error
	Listar() ([]*Usuario, error)
	BuscarPorID(id int) (*Usuario, error)
	Eliminar(id int) error
}

// =======================
// IMPLEMENTACIÓN CONCURRENTE DEL REPOSITORIO (Thread-safe con mutex)
// =======================
type repositorioMemoriaLibros struct {
	libros    map[int]*Libro
	idCounter int
	mu        sync.RWMutex // Mutex para concurrencia segura
}

func NewRepositorioMemoriaLibros() RepositorioLibros {
	return &repositorioMemoriaLibros{
		libros:    make(map[int]*Libro),
		idCounter: 1,
	}
}

func (r *repositorioMemoriaLibros) Agregar(libro *Libro) error {
	r.mu.Lock()
	defer r.mu.Unlock()

	if libro == nil {
		return errors.New("no se puede agregar un libro nil")
	}

	libro.ID = r.idCounter
	r.libros[r.idCounter] = libro
	r.idCounter++
	return nil
}

func (r *repositorioMemoriaLibros) Listar() ([]*Libro, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()

	resultado := make([]*Libro, 0, len(r.libros))
	for _, libro := range r.libros {
		// Creamos una copia para evitar exposición directa
		copia := *libro
		resultado = append(resultado, &copia)
	}
	return resultado, nil
}

func (r *repositorioMemoriaLibros) BuscarPorID(id int) (*Libro, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()

	libro, existe := r.libros[id]
	if !existe {
		return nil, ErrLibroNoEncontrado
	}
	copia := *libro
	return &copia, nil
}

func (r *repositorioMemoriaLibros) Buscar(criterio string) ([]*Libro, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()

	criterio = strings.ToLower(strings.TrimSpace(criterio))
	if criterio == "" {
		return []*Libro{}, nil
	}

	var resultados []*Libro
	for _, libro := range r.libros {
		titulo := strings.ToLower(libro.Titulo)
		autor := strings.ToLower(libro.Autor)
		if strings.Contains(titulo, criterio) || strings.Contains(autor, criterio) {
			copia := *libro
			resultados = append(resultados, &copia)
		}
	}
	return resultados, nil
}

func (r *repositorioMemoriaLibros) Actualizar(id int, libroActualizado *Libro) error {
	r.mu.Lock()
	defer r.mu.Unlock()

	if _, existe := r.libros[id]; !existe {
		return ErrLibroNoEncontrado
	}
	libroActualizado.ID = id
	r.libros[id] = libroActualizado
	return nil
}

func (r *repositorioMemoriaLibros) Eliminar(id int) error {
	r.mu.Lock()
	defer r.mu.Unlock()

	if _, existe := r.libros[id]; !existe {
		return ErrLibroNoEncontrado
	}
	delete(r.libros, id)
	return nil
}

func (r *repositorioMemoriaLibros) SiguienteID() int {
	r.mu.RLock()
	defer r.mu.RUnlock()
	return r.idCounter
}

// =======================
// SERVICIO DE NEGOCIO (Lógica de negocio separada de infraestructura)
// =======================
type GestorLibros struct {
	repo RepositorioLibros
}

func NewGestorLibros(repo RepositorioLibros) (*GestorLibros, error) {
	if repo == nil {
		return nil, ErrRepositorioNoListo
	}
	return &GestorLibros{repo: repo}, nil
}

// Métodos del servicio con validación de negocio
func (g *GestorLibros) CrearLibro(titulo, autor, categoria, formato string) (*Libro, error) {
	libro, err := NewLibro(0, titulo, autor, categoria, formato)
	if err != nil {
		return nil, fmt.Errorf("validación fallida: %w", err)
	}
	if err := g.repo.Agregar(libro); err != nil {
		return nil, fmt.Errorf("error al persistir: %w", err)
	}
	return g.repo.BuscarPorID(libro.ID)
}

func (g *GestorLibros) ObtenerLibros() ([]*Libro, error) {
	return g.repo.Listar()
}

func (g *GestorLibros) ObtenerLibroPorID(id int) (*Libro, error) {
	return g.repo.BuscarPorID(id)
}

func (g *GestorLibros) BuscarLibros(criterio string) ([]*Libro, error) {
	return g.repo.Buscar(criterio)
}

func (g *GestorLibros) ActualizarLibro(id int, titulo, autor, categoria, formato string) error {
	libroExistente, err := g.repo.BuscarPorID(id)
	if err != nil {
		return err
	}
	libroActualizado, err := NewLibro(id, titulo, autor, categoria, formato)
	if err != nil {
		return err
	}
	libroActualizado.FechaRegistro = libroExistente.FechaRegistro
	return g.repo.Actualizar(id, libroActualizado)
}

func (g *GestorLibros) EliminarLibro(id int) error {
	return g.repo.Eliminar(id)
}

// =======================
// CONTROLADORES WEB (Handlers HTTP con JSON)
// =======================
type App struct {
	gestorLibros  *GestorLibros
	gestorUsuarios interface{} // Para extensión futura
}

func NewApp(gestor *GestorLibros) *App {
	return &App{gestorLibros: gestor}
}

// Middleware para logging y manejo de errores
func loggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		log.Printf("%s %s", r.Method, r.URL.Path)
		next(w, r)
		log.Printf("Completado en %v", time.Since(start))
	}
}

// Helper para respuestas JSON
func responderJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if data != nil {
		json.NewEncoder(w).Encode(data)
	}
}

func responderError(w http.ResponseWriter, status int, mensaje string) {
	responderJSON(w, status, map[string]string{"error": mensaje})
}

// =======================
// 🌐 8 SERVICIOS WEB IMPLEMENTADOS (Requisito del proyecto)
// =======================

// 1️⃣ GET /api/libros - Listar todos los libros
func (app *App) listarLibrosHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		responderError(w, http.StatusMethodNotAllowed, "Método no permitido")
		return
	}

	// Ejecución concurrente simulada con goroutine para logging
	go func() {
		log.Printf("Solicitud de listado procesada a las %s", time.Now().Format(time.RFC3339))
	}()

	libros, err := app.gestorLibros.ObtenerLibros()
	if err != nil {
		responderError(w, http.StatusInternalServerError, "Error interno al listar libros")
		return
	}
	responderJSON(w, http.StatusOK, map[string]interface{}{
		"success": true,
		"data":    libros,
		"total":   len(libros),
	})
}

// 2️⃣ POST /api/libros - Crear nuevo libro
func (app *App) crearLibroHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		responderError(w, http.StatusMethodNotAllowed, "Método no permitido")
		return
	}

	var entrada struct {
		Titulo    string `json:"titulo"`
		Autor     string `json:"autor"`
		Categoria string `json:"categoria"`
		Formato   string `json:"formato"`
	}

	if err := json.NewDecoder(r.Body).Decode(&entrada); err != nil {
		responderError(w, http.StatusBadRequest, "JSON inválido")
		return
	}

	libro, err := app.gestorLibros.CrearLibro(
		entrada.Titulo, entrada.Autor, entrada.Categoria, entrada.Formato,
	)
	if err != nil {
		responderError(w, http.StatusBadRequest, err.Error())
		return
	}

	responderJSON(w, http.StatusCreated, map[string]interface{}{
		"success": true,
		"message": "Libro creado exitosamente",
		"data":    libro,
	})
}

// 3️⃣ GET /api/libros/{id} - Obtener libro por ID
func (app *App) obtenerLibroHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		responderError(w, http.StatusMethodNotAllowed, "Método no permitido")
		return
	}

	idStr := strings.TrimPrefix(r.URL.Path, "/api/libros/")
	id, err := strconv.Atoi(idStr)
	if err != nil || id <= 0 {
		responderError(w, http.StatusBadRequest, "ID inválido")
		return
	}

	libro, err := app.gestorLibros.ObtenerLibroPorID(id)
	if err != nil {
		if errors.Is(err, ErrLibroNoEncontrado) {
			responderError(w, http.StatusNotFound, "Libro no encontrado")
			return
		}
		responderError(w, http.StatusInternalServerError, "Error al buscar libro")
		return
	}

	responderJSON(w, http.StatusOK, map[string]interface{}{
		"success": true,
		"data":    libro,
	})
}

// 4️⃣ PUT /api/libros/{id} - Actualizar libro
func (app *App) actualizarLibroHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPut {
		responderError(w, http.StatusMethodNotAllowed, "Método no permitido")
		return
	}

	idStr := strings.TrimPrefix(r.URL.Path, "/api/libros/")
	id, err := strconv.Atoi(idStr)
	if err != nil || id <= 0 {
		responderError(w, http.StatusBadRequest, "ID inválido")
		return
	}

	var entrada struct {
		Titulo    string `json:"titulo"`
		Autor     string `json:"autor"`
		Categoria string `json:"categoria"`
		Formato   string `json:"formato"`
	}

	if err := json.NewDecoder(r.Body).Decode(&entrada); err != nil {
		responderError(w, http.StatusBadRequest, "JSON inválido")
		return
	}

	if err := app.gestorLibros.ActualizarLibro(id, entrada.Titulo, entrada.Autor, entrada.Categoria, entrada.Formato); err != nil {
		if errors.Is(err, ErrLibroNoEncontrado) {
			responderError(w, http.StatusNotFound, "Libro no encontrado")
			return
		}
		responderError(w, http.StatusBadRequest, err.Error())
		return
	}

	responderJSON(w, http.StatusOK, map[string]interface{}{
		"success": true,
		"message": "Libro actualizado exitosamente",
	})
}

// 5️⃣ DELETE /api/libros/{id} - Eliminar libro
func (app *App) eliminarLibroHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodDelete {
		responderError(w, http.StatusMethodNotAllowed, "Método no permitido")
		return
	}

	idStr := strings.TrimPrefix(r.URL.Path, "/api/libros/")
	id, err := strconv.Atoi(idStr)
	if err != nil || id <= 0 {
		responderError(w, http.StatusBadRequest, "ID inválido")
		return
	}

	if err := app.gestorLibros.EliminarLibro(id); err != nil {
		if errors.Is(err, ErrLibroNoEncontrado) {
			responderError(w, http.StatusNotFound, "Libro no encontrado")
			return
		}
		responderError(w, http.StatusInternalServerError, "Error al eliminar")
		return
	}

	responderJSON(w, http.StatusOK, map[string]interface{}{
		"success": true,
		"message": "Libro eliminado exitosamente",
	})
}

// 6️⃣ GET /api/libros/buscar?q= - Búsqueda flexible
func (app *App) buscarLibrosHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		responderError(w, http.StatusMethodNotAllowed, "Método no permitido")
		return
	}

	criterio := r.URL.Query().Get("q")
	if criterio == "" {
		responderError(w, http.StatusBadRequest, "Parámetro 'q' es requerido")
		return
	}

	resultados, err := app.gestorLibros.BuscarLibros(criterio)
	if err != nil {
		responderError(w, http.StatusInternalServerError, "Error en la búsqueda")
		return
	}

	responderJSON(w, http.StatusOK, map[string]interface{}{
		"success":  true,
		"data":     resultados,
		"total":    len(resultados),
		"criterio": criterio,
	})
}

// 7️⃣ GET /api/estadisticas - Servicio de métricas (concurrencia demostrada)
func (app *App) estadisticasHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		responderError(w, http.StatusMethodNotAllowed, "Método no permitido")
		return
	}

	// Demostración de concurrencia: múltiples goroutines procesando datos
	type Resultado struct {
		TotalLibros int
		Titulo      string
		Err         error
	}

	libros, _ := app.gestorLibros.ObtenerLibros()
	resultadosChan := make(chan Resultado, len(libros))

	// Procesamiento concurrente de cada libro (ejemplo educativo)
	for _, libro := range libros {
		go func(l *Libro) {
			// Simulación de procesamiento asíncrono
			time.Sleep(10 * time.Millisecond)
			resultadosChan <- Resultado{
				TotalLibros: len(libros),
				Titulo:      l.Titulo,
				Err:         nil,
			}
		}(libro)
	}

	// Recolección de resultados
	var titulos []string
	for i := 0; i < len(libros); i++ {
		res := <-resultadosChan
		if res.Err == nil {
			titulos = append(titulos, res.Titulo)
		}
	}

	responderJSON(w, http.StatusOK, map[string]interface{}{
		"success":        true,
		"total_libros":   len(libros),
		"titulos_procesados": titulos,
		"timestamp":      time.Now().Format(time.RFC3339),
	})
}

// 8️⃣ POST /api/libros/bulk - Carga masiva con control de concurrencia
func (app *App) cargaMasivaHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		responderError(w, http.StatusMethodNotAllowed, "Método no permitido")
		return
	}

	var entrada struct {
		Libros []struct {
			Titulo    string `json:"titulo"`
			Autor     string `json:"autor"`
			Categoria string `json:"categoria"`
			Formato   string `json:"formato"`
		} `json:"libros"`
	}

	if err := json.NewDecoder(r.Body).Decode(&entrada); err != nil {
		responderError(w, http.StatusBadRequest, "JSON inválido")
		return
	}

	// Control de concurrencia con Worker Pool (patrón útil en producción)
	const maxWorkers = 3
	tareasChan := make(chan struct {
		Titulo    string
		Autor     string
		Categoria string
		Formato   string
	}, len(entrada.Libros))
	resultadosChan := make(chan map[string]interface{}, len(entrada.Libros))

	// Workers concurrentes
	for w := 0; w < maxWorkers; w++ {
		go func() {
			for tarea := range tareasChan {
				libro, err := app.gestorLibros.CrearLibro(
					tarea.Titulo, tarea.Autor, tarea.Categoria, tarea.Formato,
				)
				if err != nil {
					resultadosChan <- map[string]interface{}{"titulo": tarea.Titulo, "error": err.Error()}
				} else {
					resultadosChan <- map[string]interface{}{"titulo": tarea.Titulo, "id": libro.ID, "success": true}
				}
			}
		}()
	}

	// Enviar tareas
	for _, l := range entrada.Libros {
		tareasChan <- struct {
			Titulo    string
			Autor     string
			Categoria string
			Formato   string
		}{l.Titulo, l.Autor, l.Categoria, l.Formato}
	}
	close(tareasChan)

	// Recoger resultados
	var resultados []map[string]interface{}
	for i := 0; i < len(entrada.Libros); i++ {
		resultados = append(resultados, <-resultadosChan)
	}

	responderJSON(w, http.StatusCreated, map[string]interface{}{
		"success":   true,
		"message":   fmt.Sprintf("Procesados %d libros", len(resultados)),
		"resultados": resultados,
	})
}

// =======================
// ROUTER Y SERVIDOR
// =======================
func (app *App) RegistrarRutas(mux *http.ServeMux) {
	// Rutas para libros (7 servicios)
	mux.HandleFunc("/api/libros", loggingMiddleware(app.listarLibrosHandler))
	mux.HandleFunc("/api/libros", loggingMiddleware(app.crearLibroHandler)) // POST
	mux.HandleFunc("/api/libros/", loggingMiddleware(func(w http.ResponseWriter, r *http.Request) {
		// Dispatcher según método HTTP
		switch r.Method {
		case http.MethodGet:
			app.obtenerLibroHandler(w, r)
		case http.MethodPut:
			app.actualizarLibroHandler(w, r)
		case http.MethodDelete:
			app.eliminarLibroHandler(w, r)
		default:
			responderError(w, http.StatusMethodNotAllowed, "Método no soportado")
		}
	}))
	mux.HandleFunc("/api/libros/buscar", loggingMiddleware(app.buscarLibrosHandler))
	mux.HandleFunc("/api/estadisticas", loggingMiddleware(app.estadisticasHandler))
	mux.HandleFunc("/api/libros/bulk", loggingMiddleware(app.cargaMasivaHandler))

	// Ruta de salud del sistema
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		responderJSON(w, http.StatusOK, map[string]string{"status": "ok", "service": "gestion-libros"})
	})
}

// =======================
// FUNCIÓN PRINCIPAL
// =======================
func main() {
	// Inicialización con inyección de dependencias
	repo := NewRepositorioMemoriaLibros()
	gestor, err := NewGestorLibros(repo)
	if err != nil {
		log.Fatalf("Error al inicializar el sistema: %v", err)
	}

	app := NewApp(gestor)
	mux := http.NewServeMux()
	app.RegistrarRutas(mux)

	// Configuración del servidor con timeouts seguros
	servidor := &http.Server{
		Addr:         ":8080",
		Handler:      mux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	log.Println("🚀 Servidor iniciado en http://localhost:8080")
	log.Println("📚 Servicios disponibles:")
	log.Println("   GET  /api/libros          → Listar libros")
	log.Println("   POST /api/libros          → Crear libro")
	log.Println("   GET  /api/libros/{id}     → Obtener libro")
	log.Println("   PUT  /api/libros/{id}     → Actualizar libro")
	log.Println("   DELETE /api/libros/{id}   → Eliminar libro")
	log.Println("   GET  /api/libros/buscar?q= → Búsqueda")
	log.Println("   GET  /api/estadisticas    → Métricas con concurrencia")
	log.Println("   POST /api/libros/bulk     → Carga masiva concurrente")

	if err := servidor.ListenAndServe(); err != nil {
		log.Fatalf("Error al iniciar servidor: %v", err)
	}
}
