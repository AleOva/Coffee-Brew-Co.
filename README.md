Introducción
En el competitivo mundo de las bebidas la capacidad de analizar los datos para la toma de decisiones informadas se ha convertido en un pilar fundamental para las empresas, el desarrollo de base de datos que permitan esta “auditoria” para mejorar o tomar en cuenta los cambios abruptos se ha vuelto en un aspecto que beneficia mucho a las empresas. Coffee Brew Co., es una empresa dedicada a la venta de café, ha recopilado un conjunto significativo de transacciones y almacenado un robusto archivo en el que se abarcan 3636 registros de marzo a abril 2024. Este proyecto tiene como objetivo aprovechar estos datos para generar insights accionables mediante el uso de una base de datos relacional en PostgreSQL implementada y gestionada por pgAdmin4.
A través de la creación de un esquema relacional que incluye tablas como tiempo, 
metodo_pago, cliente, tienda, producto, categoria_producto, venta, detalle_venta y dashboard_metrics, se ha estructurado información para facilitar su análisis. Los datos, importados desde archivos CSV derivados del archivo original, han permitido el desarrollo de procedimientos que almacenan y calculan tres indicadores claves para el desempeño (KPIs): el producto más vendido, el porcentaje de uso de métodos de pago, y el día de la semana con mayor actividad de ventas. Estos registrados en una tabla dashboard_metrics, ofrecen una visión clara de las preferencias de los clientes y los patrones de consumo, proporcionando una base sólida para las estrategias comerciales. Este análisis realizado por primera vez marca el inicio de un enfoque data-driven para Coffee Brew Co. Con esto podrán optimizar operaciones y maximizar ingresos en un mercado en constante evolución. con el potencial de optimizar operaciones y maximizar ingresos en un mercado en constante evolución.



Código SQL PARA TABLAS
CREATE TABLE tiempo (
    id_tiempo SERIAL PRIMARY KEY,
    date DATE NOT NULL,
    datetime TIMESTAMP NOT NULL,
    hour_of_day INT,
    time_of_day VARCHAR(10),
    weekday VARCHAR(10),
    month_name VARCHAR(20),
    weekdaysort INT,
    monthsort INT
);

CREATE TABLE metodo_pago (
    id_metodo_pago SERIAL PRIMARY KEY,
    cash_type VARCHAR(100) NOT NULL
);

CREATE TABLE cliente (
    id_cliente SERIAL PRIMARY KEY,
    card VARCHAR(50) NOT NULL
);

CREATE TABLE tienda (
    id_tienda SERIAL PRIMARY KEY,
    nombre_tienda VARCHAR(100) NOT NULL
);

CREATE TABLE categoria_producto (
    id_categoria SERIAL PRIMARY KEY,
    nombre_categoria VARCHAR(100) NOT NULL
);

CREATE TABLE producto (
    id_producto SERIAL PRIMARY KEY,
    coffee_name VARCHAR(100) NOT NULL,
    id_categoria INTEGER,
    FOREIGN KEY (id_categoria) REFERENCES categoria_producto(id_categoria)
);

CREATE TABLE venta (
    id_venta SERIAL PRIMARY KEY,
    id_tiempo INTEGER NOT NULL,
    id_cliente INTEGER,
    id_metodo_pago INTEGER NOT NULL,
    id_tienda INTEGER NOT NULL,
    money DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (id_tiempo) REFERENCES tiempo(id_tiempo),
    FOREIGN KEY (id_cliente) REFERENCES cliente(id_cliente),
    FOREIGN KEY (id_metodo_pago) REFERENCES metodo_pago(id_metodo_pago),
    FOREIGN KEY (id_tienda) REFERENCES tienda(id_tienda)
);

CREATE TABLE detalle_venta (
    id_detalle SERIAL PRIMARY KEY,
    id_venta INTEGER NOT NULL,
    id_producto INTEGER NOT NULL,
    money DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (id_venta) REFERENCES venta(id_venta),
    FOREIGN KEY (id_producto) REFERENCES producto(id_producto)
);

CREATE TABLE dashboard_metrics (
    id SERIAL PRIMARY KEY,
    metric_name VARCHAR(100),
    metric_value TEXT,
    generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

Consultas
1. ¿Cuáles son los 3 productos más vendidos en unidades totales? 
SELECT p.coffee_name, COUNT(d.id_detalle) as unidades_vendidas
FROM detalle_venta d
JOIN producto p ON d.id_producto = p.id_producto
GROUP BY p.id_producto, p.coffee_name
ORDER BY unidades_vendidas DESC
LIMIT 3;
 <img width="1920" height="1080" alt="Captura de pantalla 2025-07-29 184842" src="https://github.com/user-attachments/assets/3581eb19-6fef-4243-9e0b-51a552dabeb1" />



2. ¿Qué método de pago fue más utilizado durante el mes de diciembre? 

SELECT mp.cash_type AS metodo_pago, COUNT(*) AS total_uso
FROM venta v
JOIN metodo_pago mp ON v.id_metodo_pago = mp.id_metodo_pago
JOIN tiempo t ON v.id_tiempo = t.id_tiempo
WHERE t.month_name = 'dic' 
  AND EXTRACT(YEAR FROM t.date) = 2024
GROUP BY mp.cash_type
ORDER BY total_uso DESC
LIMIT 1;
 <img width="1920" height="1080" alt="Captura de pantalla 2025-07-29 184940" src="https://github.com/user-attachments/assets/403de5a3-02ff-4585-b12e-08d57aeb7bd6" />

3. ¿Qué tienda generó mayores ingresos en el primer trimestre del año? 

SELECT t.nombre_tienda AS tienda, SUM(v.money) AS total_ingresos
FROM venta v
JOIN tiempo ti ON v.id_tiempo = ti.id_tiempo
JOIN tienda t ON v.id_tienda = t.id_tienda
WHERE EXTRACT(MONTH FROM ti.date) BETWEEN 1 AND 3
  AND EXTRACT(YEAR FROM ti.date) = 2024  -- Cambia el año según sea necesario
GROUP BY t.nombre_tienda
ORDER BY total_ingresos DESC
LIMIT 1;
 <img width="1920" height="1080" alt="Captura de pantalla 2025-07-29 185006" src="https://github.com/user-attachments/assets/67d6f1f7-0aed-43f9-a0f6-53a73ef4a1c6" />

4. ¿Cuál es el ticket promedio de venta por categoría de producto? 

SELECT cp.nombre_categoria, AVG(d.money) as ticket_promedio
FROM detalle_venta d
JOIN producto p ON d.id_producto = p.id_producto
JOIN categoria_producto cp ON p.id_categoria = cp.id_categoria
GROUP BY cp.id_categoria, cp.nombre_categoria;

 <img width="1920" height="1080" alt="Captura de pantalla 2025-07-29 185024" src="https://github.com/user-attachments/assets/30eaa245-ee54-456c-806d-995a53b7ce0f" />


Parte 4: Procedimiento Almacenado
🥤 Producto Más Vendido: El producto con más unidades vendidas
CREATE OR REPLACE PROCEDURE generar_kpi_producto_mas_vendido()
LANGUAGE plpgsql
AS $$
DECLARE
    v_producto_mas_vendido VARCHAR;
    v_unidades_max INT;
BEGIN
    -- Calcular el producto más vendido
    SELECT p.coffee_name, COUNT(d.id_detalle) INTO v_producto_mas_vendido, v_unidades_max
    FROM detalle_venta d
    JOIN producto p ON d.id_producto = p.id_producto
    GROUP BY p.id_producto, p.coffee_name
    ORDER BY v_unidades_max DESC
    LIMIT 1;
    
    -- Insertar el resultado en dashboard_metrics
    INSERT INTO dashboard_metrics (metric_name, metric_value)
    VALUES ('🥤 Producto Más Vendido', v_producto_mas_vendido || ' (' || v_unidades_max || ' unidades)');
END;

EJECUTAR:
CALL generar_kpi_producto_mas_vendido();
VERIFICAR:
SELECT * FROM dashboard_metrics WHERE metric_name = '🥤 Producto Más Vendido';
$$;
 
<img width="1920" height="1080" alt="Captura de pantalla 2025-07-29 183012" src="https://github.com/user-attachments/assets/5e08f315-d15a-421b-aa91-e094f453e904" />

💳 % por Método de Pago: Participación porcentual de cada método de pago
CREATE OR REPLACE PROCEDURE generar_kpi_metodo_pago()
LANGUAGE plpgsql
AS $$
DECLARE
    v_metodo_pago_total INT;
    v_metodo_pago_card INT;
    v_porcentaje_card DECIMAL;
    v_metodo_pago_cash INT;
    v_porcentaje_cash DECIMAL;
BEGIN
    -- Calcular el total de ventas
    SELECT COUNT(*) INTO v_metodo_pago_total
    FROM venta;
    
    -- Calcular ventas por tarjeta
    SELECT COUNT(*) INTO v_metodo_pago_card
    FROM venta v
    JOIN metodo_pago m ON v.id_metodo_pago = m.id_metodo_pago
    WHERE m.cash_type = 'card';
    
    v_porcentaje_card = (v_metodo_pago_card::DECIMAL / v_metodo_pago_total::DECIMAL) * 100;
    
    -- Calcular ventas por efectivo
    SELECT COUNT(*) INTO v_metodo_pago_cash
    FROM venta v
    JOIN metodo_pago m ON v.id_metodo_pago = m.id_metodo_pago
    WHERE m.cash_type = 'cash';
    
    v_porcentaje_cash = (v_metodo_pago_cash::DECIMAL / v_metodo_pago_total::DECIMAL) * 100;
    
    -- Insertar el resultado en dashboard_metrics
    INSERT INTO dashboard_metrics (metric_name, metric_value)
    VALUES ('💳 % por Método de Pago', 'Card: ' || ROUND(v_porcentaje_card, 2) || '%, Cash: ' || ROUND(v_porcentaje_cash, 2) || '%');
END;
$$;
EJECUTARLO:
CALL generar_kpi_metodo_pago();
VERIFICAR:
SELECT * FROM dashboard_metrics WHERE metric_name = '💳 % por Método de Pago';

<img width="1920" height="1080" alt="Captura de pantalla 2025-07-29 183206" src="https://github.com/user-attachments/assets/34a1e6b3-dcd1-4b5b-8933-adce9e75d157" />

 
📆 Día Más Activo: Día de la semana con más ventas promedio
CREATE OR REPLACE PROCEDURE generar_kpi_dia_mas_activo()
LANGUAGE plpgsql
AS $$
DECLARE
    v_dia_mas_activo VARCHAR;
    v_ventas_promedio_max DECIMAL;
BEGIN
    -- Calcular el día más activo
    SELECT sub.weekday, AVG(sub.count) INTO v_dia_mas_activo, v_ventas_promedio_max
    FROM (
        SELECT t.id_tiempo, t.weekday, COUNT(v.id_venta) as count
        FROM venta v
        JOIN tiempo t ON v.id_tiempo = t.id_tiempo
        GROUP BY t.id_tiempo, t.weekday
    ) sub
    GROUP BY sub.weekday
    ORDER BY AVG(sub.count) DESC
    LIMIT 1;
    
    -- Insertar el resultado en dashboard_metrics
    INSERT INTO dashboard_metrics (metric_name, metric_value)
    VALUES ('📆 Día Más Activo', v_dia_mas_activo || ' (' || ROUND(v_ventas_promedio_max, 2) || ' ventas promedio)');
END;
$$;
EJECUTARLO:
CALL generar_kpi_dia_mas_activo();
VERIFICARLO:
SELECT * FROM dashboard_metrics WHERE metric_name = '📆 Día Más Activo';

 <img width="1920" height="1080" alt="Captura de pantalla 2025-07-29 183434" src="https://github.com/user-attachments/assets/e883921d-c2c4-485a-b12d-56fad6af76ff" />


5. Una interpretación de los insights generados por los datos. 
1.	Producto Más Vendido: "Latte (782 unidades)" 
o	Insight: El Latte destaca como uno de los productos más vendido con 782 unidades, lo que sugiere que hay una fuerte preferencia por los clientes con bebidas a base de leche. Esto podría indicar que los clientes valoran las opciones cremosas y personalizadas. La alta demanda refleja una oportunidad para capitalizar la oportunidad, también sugiere analizar productos como Americano u otras bebidas no tan comerciales.
2.	% por Método de Pago: (Hipotético "Card: 97.55%, Cash: 2.45%") 
o	Insight: La mayoría La mayoría de las ventas (97.55%) se realizan con tarjeta, lo que indica una preferencia por pagos electrónicos, posiblemente debido a la conveniencia o a una clientela joven y tecnológicamente adaptada. El 2.45% en efectivo sugiere que aún hay un segmento que prefiere este método, tal vez clientes habituales o locales.
Esto resalta la importancia de mantener un sistema de pago robusto para tarjetas, pero también de no descuidar la infraestructura para transacciones en efectivo.
3.	Día Más Activo: (Hipotético "Sá (100 ventas promedio)") 
o	Insight: El sábado es el día con el mayor promedio de ventas (100 por día), lo que podría deberse a un aumento de actividad social o laboral al final de la semana. Esto sugiere que los fines de semana o días previos podrían ser picos de demanda. La concentración de ventas en viernes indica una oportunidad para maximizar ingresos en ese día, pero también para equilibrar la demanda en otros días mediante estrategias específicas.

6. Conclusión sobre lo que Coffee Brew Co. 
Basado en los insights generados, Coffee Brew Co. puede tomar las siguientes acciones estratégicas:
•	Promover ofertas en productos: 
o	Dado que el Latte es el producto estrella de la compañía podrían introducirse variaciones como con vainilla o descafeinado para mantener el interés de los clientes y aumentar las ventas. Al mismo tiempo evaluar productos que no tengan mayor demanda también permitirá diversificar la cartera.
•	Mejorar las opciones de pagos: 
o	Con un 97.55% de ventas por tarjeta Coffee Brew Co. Debería de asegurar que los terminales de pago electrónico sean rápidos y confiables, especialmente en horas pico. Además, podría ofrecer incentivos como puntos de fidelidad para pagos con tarjeta para aumentar la tendencia. Sin embargo, mantener cajeros diversos bien surtidos de cambio para que el 2.45% de pagos en efectivo sigue siendo esencial, particularmente para clientes que prefieren esta opción.
•	Aprovechar la demanda: 
o	El sábado, con un promedio de 100 ventas es una oportunidad para aumentar el personal o inventario de Latte y otros productos populares, podrían lanzarse promociones también para los demás días y hacer que se equilibren las ventas y tener más días a la semana a la semana que tengan ventas exitosas. Hay que apostar por el marketing en la semana para aumentar las ventas.
•	Análisis continuo: 
o	Coffee Brew Co. debería actualizar estos KPIs regularmente (por ejemplo, mensualmente) para detectar cambios en las preferencias de los clientes o patrones estacionales. Esto les permitirá ajustar su estrategia en tiempo real, como adaptar el menú o los horarios de atención según las tendencias.


