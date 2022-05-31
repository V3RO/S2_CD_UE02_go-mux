# In-Class Exercise 02 (go-mux)
[![Build](https://github.com/V3RO/S2_CD_UE02_go-mux/actions/workflows/go.yml/badge.svg)]([https://github.com/V3RO/S2_CD_UE02_go-mux](https://github.com/V3RO/S2_CD_UE02_go-mux)/actions/workflows/go.yml)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=V3RO_S2_CD_UE02_go-mux&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=V3RO_S2_CD_UE02_go-mux)
## Implementation

### Docker

For the PostgreSQL database I created a docker-compose file, which can be found [here](./docker/docker-compose.yaml). <br>
When creating the container the database will be initialized with a product table. The init script can be found [here](./docker/sql/init.sql) <br>

### Go
For the implementation I followed the provided guide.

The [main](./src/main.go) established the databse connection and defines the available endpoints. <br>
The [model](./src/model.go) contains the product model definition and the available methods. <br>
The [main_test](./src/main_test.go) contains the tests for the project. <br>


For the custom features I added the following endpoints:
- get products by name
- clear/delete all products

The `getProductsByName` receives a string containing the name that will be searched for. 
The query is just a simple SELECT, to make it case insensitive the name gets transformed to lower case.

```go
func getProductsByName(db *sql.DB, name string) ([]product, error) {
	rows, err := db.Query(
		"SELECT id, name, price FROM products WHERE lower(name) LIKE '%' || lower($1) || '%'", name)

	if err != nil {
		return nil, err
	}

	defer rows.Close()

	products := []product{}

	for rows.Next() {
		var p product
		if err := rows.Scan(&p.ID, &p.Name, &p.Price); err != nil {
			return nil, err
		}
		products = append(products, p)
	}

	return products, nil
}

```

The `clear` is just a simple truncate, which removes all products from the table, without deleting the table itself.

```go
func clear(db *sql.DB) error {
	_, err := db.Exec("TRUNCATE products")

	return err
}

```

To make both functions available as endpoints they get added to the `initializeRoutes` which defines the route of the endpoint and which function should be called.
Since the clear endpoint does not return anything I added another helper function similiar to the ones from the tutorial which respondes with an empty body.
```go
func (a *App) initializeRoutes() {
	a.Router.HandleFunc("/products", a.clear).Methods("DELETE")
	a.Router.HandleFunc("/product/{name}", a.getProductsByName).Methods("GET")
}

func (a *App) clear(w http.ResponseWriter, r *http.Request) {

	err := clear(a.DB)
	if err != nil {
		respondWithError(w, http.StatusInternalServerError, err.Error())
		return
	}

	respondWithEmptyBody(w, http.StatusOK)
}

func (a *App) getProductsByName(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	name := vars["name"]

	products, err := getProductsByName(a.DB, name)
	if err != nil {
		respondWithError(w, http.StatusInternalServerError, err.Error())
		return
	}

	respondWithJSON(w, http.StatusOK, products)
}

func respondWithEmptyBody(w http.ResponseWriter, code int) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(code)
}
```

The newly added endpoints should also get tested. Therefore I added the following two tests and another helper function which checks if the right amount of items get returned from the endpoints.

```go
func TestClear(t *testing.T) {
	clearTable()
	addProducts(5)
	var products []map[string]interface{}

	req, _ := http.NewRequest("GET", "/products", nil)
	response := executeRequest(req)
	checkResponseCode(t, http.StatusOK, response.Code)

	json.Unmarshal(response.Body.Bytes(), &products)
	checkResponseItemCount(t, 5, len(products))

	req, _ = http.NewRequest("DELETE", "/products", nil)
	response = executeRequest(req)
	checkResponseCode(t, http.StatusOK, response.Code)

	req, _ = http.NewRequest("GET", "/products", nil)
	response = executeRequest(req)
	checkResponseCode(t, http.StatusOK, response.Code)

	json.Unmarshal(response.Body.Bytes(), &products)
	checkResponseItemCount(t, 0, len(products))
}

func TestGetProductsByName(t *testing.T) {
	clearTable()
	addProducts(5)
	var products []map[string]interface{}

	req, _ := http.NewRequest("GET", "/product/product", nil)
	response := executeRequest(req)
	checkResponseCode(t, http.StatusOK, response.Code)

	json.Unmarshal(response.Body.Bytes(), &products)
	checkResponseItemCount(t, 5, len(products))

	req, _ = http.NewRequest("GET", "/product/product 1", nil)
	response = executeRequest(req)
	checkResponseCode(t, http.StatusOK, response.Code)

	json.Unmarshal(response.Body.Bytes(), &products)
	checkResponseItemCount(t, 1, len(products))
}

func checkResponseItemCount(t *testing.T, expected, actual int) {
	if expected != actual {
		t.Errorf("Expected %d items. Got %d\n", expected, actual)
	}
}
```
