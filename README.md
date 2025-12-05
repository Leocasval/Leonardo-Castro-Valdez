<?php
error_reporting(E_ALL);
ini_set('display_errors', 1);

session_start();
require "db.php";

// Aqui esta los mensaje temporales 

$mensaje = "";
if (isset($_SESSION["mensaje"])) {
    $mensaje = $_SESSION["mensaje"];
    unset($_SESSION["mensaje"]);
}
R
// Aqui se obtiene la categoria para saber en que categoria pertenece el producto

$categorias = $pdo->query("SELECT * FROM categorias")->fetchAll(PDO::FETCH_ASSOC);

// Busqueda y filtro buscas el producto por busqueda o si no por los filtros o categoria 

$buscar = $_GET["buscar"] ?? "";
$categoria = $_GET["categoria"] ?? "";

$query = "SELECT * FROM productos WHERE 1";
$params = [];

if ($buscar !== "") {
    $query .= " AND nombre LIKE :b";
    $params[":b"] = "%$buscar%";
}

if ($categoria !== "") {
    $query .= " AND categoria_id = :cat";
    $params[":cat"] = $categoria;
}

$stmt = $pdo->prepare($query);
$stmt->execute($params);
$productos = $stmt->fetchAll(PDO::FETCH_ASSOC);

// Aqui esta para agregar a la bolsa de los productos a llevar 

if (isset($_GET["accion"]) && $_GET["accion"] == "agregar") {
    $id = $_GET["id"];

    if (!isset($_SESSION["bolsa"])) {
        $_SESSION["bolsa"] = [];
    }

    if (isset($_SESSION["bolsa"][$id])) {
        $_SESSION["bolsa"][$id]["cantidad"]++;
    } else {
        $stmt = $pdo->prepare("SELECT * FROM productos WHERE id = ?");
        $stmt->execute([$id]);
        $p = $stmt->fetch(PDO::FETCH_ASSOC);

        if ($p) {
            $_SESSION["bolsa"][$id] = [
                "id" => $p["id"],
                "nombre" => $p["nombre"],
                "precio" => $p["precio"],
                "imagen" => $p["imagen"],
                "cantidad" => 1
            ];
        }
    }

    $_SESSION["mensaje"] = "Producto agregado a la bolsa 🛍️";
    header("Location: supermercado.php");
    exit;
}

// Para elimina producto por producto 

if (isset($_GET["accion"]) && $_GET["accion"] == "eliminar") {
    $id = $_GET["id"];
    unset($_SESSION["bolsa"][$id]);

    $_SESSION["mensaje"] = "Producto eliminado con éxito ❌";
    header("Location: supermercado.php");
    exit;
}

// Aqui esta para cambiar la cantidad de productos

if (isset($_POST["accion"]) && $_POST["accion"] == "cambiar") {
    $id = $_POST["id"];
    $cantidad = intval($_POST["cantidad"]);

    if ($cantidad > 0) {
        $_SESSION["bolsa"][$id]["cantidad"] = $cantidad;
        $_SESSION["mensaje"] = "Cantidad actualizada ✔️";
    }

    header("Location: supermercado.php");
    exit;
}

// Para vaciar la bolsa de todo los productos 

if (isset($_GET["accion"]) && $_GET["accion"] == "vaciar") {
    unset($_SESSION["bolsa"]);
    $_SESSION["mensaje"] = "Bolsa vaciada 🗑️";
    header("Location: supermercado.php");
    exit;
}

// Aqui se ecuentra la opcion de pagar 

if (isset($_GET["accion"]) && $_GET["accion"] == "pagar") {
    unset($_SESSION["bolsa"]);
    $_SESSION["mensaje"] = "Pago exitoso. ¡Gracias por tu compra! 🎉";
    header("Location: supermercado.php");
    exit;
}

// El contador de la cantidad de producto a llevar 
$bolsa_count = 0;
if (isset($_SESSION["bolsa"])) {
    foreach ($_SESSION["bolsa"] as $item) {
        $bolsa_count += $item["cantidad"];
    }
}
?>
<!DOCTYPE html>
<html>
<head>
    <title>Megashop24</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-100">

<!-- Un pequeño mensaje  -->
<?php if ($mensaje): ?>
<div class="p-4 bg-green-500 text-white text-center">
    <?= $mensaje ?>
</div>
<?php endif; ?>

<!-- Encabezado -->
<div class="bg-blue-600 p-4 text-white flex justify-between items-center">
    <h1 class="text-2xl font-bold">Megashop24</h1>

    <a href="?accion=ver_bolsa" class="text-xl">
        🛍️ Bolsa (<?= $bolsa_count ?>)
    </a>
</div>

<!-- Buscador mas el filtro de busqueda por categoria -->
<div class="p-4">
    <form method="GET" class="flex gap-4">
        <input type="text" name="buscar" placeholder="Buscar producto..."
               value="<?= $buscar ?>"
               class="p-2 border rounded w-1/2">

        <select name="categoria" class="p-2 border rounded">
            <option value="">Todas las categorías</option>
            <?php foreach ($categorias as $c): ?>
                <option value="<?= $c['id'] ?>" <?= $categoria == $c["id"] ? "selected" : "" ?>>
                    <?= $c["nombre"] ?>
                </option>
            <?php endforeach; ?>
        </select>

        <button class="bg-blue-600 text-white px-4 py-2 rounded">Filtrar</button>
    </form>
</div>

<!-- ¨Los productos que ofrece la pagina  -->
<div class="grid grid-cols-1 md:grid-cols-3 lg:grid-cols-4 gap-4 p-4">
    <?php foreach ($productos as $p): ?>
        <div class="bg-white p-4 shadow rounded">
            <img src="img/<?= $p["imagen"] ?>" class="h-40 w-full object-cover rounded">
            <h2 class="text-lg font-bold mt-2"><?= $p["nombre"] ?></h2>
            <p class="text-green-600 font-bold">$<?= $p["precio"] ?></p>

            <a href="?accion=agregar&id=<?= $p["id"] ?>"
               class="mt-2 block bg-blue-500 text-white text-center py-2 rounded">
               Agregar 🛍️
            </a>
        </div>
    <?php endforeach; ?>
</div>

<!-- La bolsa que es el carrito donde se aguarda las compras  -->
<?php if (isset($_GET["accion"]) && $_GET["accion"] == "ver_bolsa"): ?>

<div class="p-4 bg-white shadow rounded w-3/4 mx-auto">

    <h2 class="text-2xl font-bold mb-4">🛍️ Tu Bolsa</h2>

    <?php if (empty($_SESSION["bolsa"])): ?>
        <p class="text-gray-600">Tu bolsa está vacía</p>
    <?php else: ?>

    <table class="w-full">
        <tr class="font-bold border-b">
            <td>Producto</td>
            <td>Precio</td>
            <td>Cantidad</td>
            <td>Total</td>
            <td></td>
        </tr>

        <?php
        $total = 0;
        foreach ($_SESSION["bolsa"] as $p):
            $sub = $p["precio"] * $p["cantidad"];
            $total += $sub;
        ?>
        <tr class="border-b">
            <td class="flex items-center gap-2 p-2">
                <img src="img/<?= $p["imagen"] ?>" class="h-12 w-12 object-cover rounded">
                <?= $p["nombre"] ?>
            </td>

            <td>$<?= $p["precio"] ?></td>

            <td>
                <form method="POST">
                    <input type="hidden" name="accion" value="cambiar">
                    <input type="hidden" name="id" value="<?= $p["id"] ?>">

                    <input type="number" name="cantidad" value="<?= $p["cantidad"] ?>"
                           class="border p-1 w-16">
                    <button class="bg-blue-500 text-white px-2 py-1 rounded">✔️</button>
                </form>
            </td>

            <td>$<?= $sub ?></td>

            <td>
                <a href="?accion=eliminar&id=<?= $p["id"] ?>"
                   class="text-red-500 font-bold">X</a>
            </td>
        </tr>
        <?php endforeach; ?>
    </table>

    <h3 class="text-xl font-bold mt-4">Total: $<?= $total ?></h3>

    <div class="flex gap-4 mt-4">
        <a href="?accion=vaciar" class="bg-red-500 text-white px-4 py-2 rounded">Vaciar bolsa</a>
        <a href="?accion=pagar" class="bg-green-600 text-white px-4 py-2 rounded">Pagar</a>
    </div>

    <?php endif; ?>

</div>

<?php endif; ?>

</body>
</html>


<?php
$host = "localhost";
$user = "root";
$pass = "";
$dbname = "supermercado";

try {
    $pdo = new PDO(
        "mysql:host=$host;dbname=$dbname;charset=utf8mb4",
        $user,
        $pass
    );
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch (PDOException $e) {
    die("❌ Error BD: " . $e->getMessage());
}
?>

CREATE DATABASE IF NOT EXISTS supermercado;
USE supermercado;

CREATE TABLE categorias (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(50) NOT NULL
);

INSERT INTO categorias (nombre) VALUES
('Bebidas'),
('Lácteos'),
('Panadería'),
('Frutas y Verduras'),
('Limpieza'),
('Dulces'),
('Higiene Personal'),
('Carnes');

CREATE TABLE productos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(150) NOT NULL,
    precio DECIMAL(10,2) NOT NULL,
    imagen VARCHAR(255) NOT NULL,
    categoria_id INT NOT NULL,
    FOREIGN KEY (categoria_id) REFERENCES categorias(id)
);

INSERT INTO productos (nombre, precio, imagen, categoria_id) VALUES
('Coca Cola 600ml', 18.50, 'coca.jpg', 1),
('Pepsi 600ml', 17.00, 'pepsi.jpg', 1),
('Jugo Jumex Durazno', 13.50, 'jumex.jpg', 1),

('Leche Alpura 1L', 27.00, 'leche.jpg', 2),
('Yogurt Natural Lala', 12.00, 'yogurt.jpg', 2),

('Pan Blanco Bimbo', 35.00, 'pan.jpg', 3),
('Pan Integral Bimbo', 38.00, 'pan_integral.jpg', 3),

('Manzana Roja (kg)', 32.00, 'manzana.jpg', 4),
('Plátano (kg)', 28.00, 'platano.jpg', 4),

('Cloro 1L', 22.00, 'cloro.jpg', 5),
('Jabón Roma 1kg', 25.00, 'roma.jpg', 5),

('Chocolate Hersheys', 11.00, 'hersheys.jpg', 6),
('Gomitas Trululu', 8.00, 'gomitas.jpg', 6),

('Shampoo Sedal 750ml', 34.00, 'shampoo.jpg', 7),
('Jabón de baño Dove', 19.00, 'dove.jpg', 7),

('Pollo entero (kg)', 65.00, 'pollo.jpg', 8),
('Carne molida (kg)', 82.00, 'carne_molida.jpg', 8);

CREATE TABLE carrito (
    id INT AUTO_INCREMENT PRIMARY KEY,
    producto_id INT,
    cantidad INT,
    fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (producto_id) REFERENCES productos(id)
);
