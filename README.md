# cloud1_0
Alimentar un repositorio de información del Padrón Reducido SUNAT alojado en una base de datos ORACLE con APIS y uso de JAVA para el consumo desde una aplicación híbrida(Apache Cordova) usando comunicación SOAP(PHP).

También demostraremos cómo utilizar dicha información para generar un JSON que permita la GEOLOCALIZACIÓN del Padrón Reducido

### 1.- Conseguir información complementaria (WORKSTATION)
Crearemos la siguiente clase .java que permita obtener informacion necesaria(Latitud y  Longitud) de direcciones del Padrón Reducido e insertarla en la Base de Datos ORACLE XE

```JAVA
/*
 * To change this license header, choose License Headers in Project Properties.
 * To change this template file, choose Tools | Templates
 * and open the template in the editor.
 */
package oraclelatlon;

//IMPORTAREMOS LAS LIBRERIAS NECESARIAS PARA CONECTIVIDAD BD Y MANEJO DE JSON (Maven Repository)
import java.io.BufferedReader;
import java.io.FileWriter;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.Reader;
import java.net.MalformedURLException;
import java.net.URL;
import java.nio.charset.Charset;
import java.sql.DriverManager;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import org.json.JSONException;
import org.json.JSONObject;
import org.apache.commons.lang.StringUtils;


public class OracleLatLon {

    private static final String DB_DRIVER = "oracle.jdbc.driver.OracleDriver";
    private static final String DB_CONNECTION = "jdbc:oracle:thin:@IP_BD:PUERTO_BD:INSTANCIA_BD";
    private static final String DB_USER = "USUARIO";
    private static final String DB_PASSWORD = "CONTRASEÑA";

    public static void main(String[] argv) throws IOException {

        try {

            selectRecordsFromDbUserTable();

        } catch (SQLException e) {

            System.out.println(e.getMessage());

        }

    }

    private static void WriteTxt(String texto) {
        try {
            FileWriter writer = new FileWriter("C:\\Users\\Person\\Documents\\ubigeo\\Distritos\\Magdalena.txt", true);
            writer.write(texto);
            writer.write("\r\n");   // write new line
            writer.close();
        } catch (IOException e) {

        }
    }

    private static String readAll(Reader rd) throws IOException {
        StringBuilder sb = new StringBuilder();
        int cp;
        while ((cp = rd.read()) != -1) {
            sb.append((char) cp);
        }
        return sb.toString();
    }

    public static JSONObject readJsonFromUrl(String url) throws IOException, JSONException {
        InputStream is = new URL(url).openStream();
        try {
            BufferedReader rd = new BufferedReader(new InputStreamReader(is, Charset.forName("UTF-8")));
            String jsonText = readAll(rd);
            JSONObject json = new JSONObject(jsonText);
            return json;
        } finally {
            is.close();
        }
    }

    private static void selectRecordsFromDbUserTable() throws SQLException, MalformedURLException, IOException {

        Connection dbConnection = null;
        Statement statement = null;

//GENERAREMOS EL SELECT QUE TRAIGA LAS DIRECCIONES PARA LA AUTOMATIZACIÓN DE OBTENCION DE COORDENADAS

        String selectTableSQL = "SELECT  PR.RUC, O.DISTRITO, PR.TIPO_VIA, PR.NOMBRE_VIA, PR.NUMERO FROM PADRON_REDUCIDO PR JOIN UBIGEO O ON PR.ID_UBIGEO=O.ID_UBIGEO WHERE  PR.NOMBRE_VIA != '-' and PR.NUMERO !='-' and PR.NOMBRE_VIA != '---' and DISTRITO='MAGDALENA DEL MAR' and LATITUD IS NULL";
        String insertTableSQL = "";
        try {
            dbConnection = getDBConnection();
            statement = dbConnection.createStatement();

            //System.out.println(selectTableSQL);
            // execute select SQL stetement
            ResultSet rs = statement.executeQuery(selectTableSQL);
            int conteo = 0;

            while (rs.next() & conteo<2000) {

                //ALMACENAMOS CADA CELDA DE LA FILA EN UNA VARIABLE
                String RUC = rs.getString("RUC");
                String DISTRITO = rs.getString("DISTRITO");
                String TIPO_VIA = rs.getString("TIPO_VIA");
                String NOMBRE_VIA = rs.getString("NOMBRE_VIA");
                String NUMERO = rs.getString("NUMERO");

                //CONCATENAMOS E INVOCAMOS A MAPS
                String url = "https://maps.googleapis.com/maps/api/geocode/json?address=" /*+ TIPO_VIA + "+"*/ + NOMBRE_VIA + "+" + NUMERO + ",+" + DISTRITO;
                url = url.replace(" ", "+");//Jirón+Larco+Herrera+665,+Magdalena+del+Mar";
                url += "&key=AIzaSyCwkZCN8d463YQ2W45mxrKSlBvs0Z5kEMw";
                JSONObject json = readJsonFromUrl(url);

                String respuesta = json.toString();

                if (respuesta.indexOf("location\":{\"lng\":", 1) == -1) {
                    System.out.println("No se pudo RUC: " + RUC);
                } else {
                    String cadena = respuesta.substring(respuesta.indexOf("location\":{\"lng\":", 1));
                    System.out.println(cadena);
                    String latlon = cadena.substring(cadena.indexOf("\"lng\":", 1));
                    //System.out.println(latlon);
                    String longitud = StringUtils.substringBetween(latlon, "\"lng\":", ",");
                    String latitud = StringUtils.substringBetween(latlon, "\"lat\":", "}");
 
 //ESCRIBIREMOS EN UN .TXT TODAS LAS SENTENCIAS NECESARIAS PARA DESPUÉS INVOCAR AL ARCHIVO DEDE SQLPLUS
                    insertTableSQL = "UPDATE PADRON_REDUCIDO SET LATITUD='" + latitud.replace("]", "") + "', LONGITUD='" + longitud + "' WHERE RUC='" + RUC + "';";
                    WriteTxt(insertTableSQL);

                }
                conteo++;
            }

        } catch (SQLException e) {

            System.out.println(e.getMessage());

        } finally {

            if (statement != null) {
                statement.close();
            }

            if (dbConnection != null) {
                dbConnection.close();
            }

        }

    }

    private static Connection getDBConnection() {

        Connection dbConnection = null;

        try {

            Class.forName(DB_DRIVER);

        } catch (ClassNotFoundException e) {

            System.out.println(e.getMessage());

        }

        try {

            dbConnection = DriverManager.getConnection(DB_CONNECTION, DB_USER,
                    DB_PASSWORD);
            return dbConnection;

        } catch (SQLException e) {

            System.out.println(e.getMessage());

        }

        return dbConnection;

    }

}
``` 

### 2.- Crear SOAP WEBSERVICE en PHP para conseguir la información (WEBSERVER)
Crearemos la siguiente clase .php que apunte a las columnas de nuestra base de datos Oracle XE

```PHP
<?php

require_once "lib/nusoap.php";
// Create SOAP Server
$server = new soap_server();
$server->configureWSDL("Test_Service", "http://www.example.com/test_service");

function GET_PADRON_DIRECCIONES($P_DISTRITO) {

    $conn = oci_connect('@USUARIO_ORACLE', '@CONTRASEÑA_ORACLE', '@IP_ORACLE/@INSTANCIA_ORACLE');
    if (!$conn) {
        $e = oci_error();
        trigger_error(htmlentities($e['message'], ENT_QUOTES), E_USER_ERROR);
    }
    $sql = 'SELECT convert(PR.ID_LOTE,\'US7ASCII\',\'WE8ISO8859P1\') AS ID_LOTE,convert(PR.RUC,\'US7ASCII\',\'WE8ISO8859P1\')AS RUC,convert(PR.RAZON_SOCIAL,\'US7ASCII\',\'WE8ISO8859P1\')AS RAZON_SOCIAL,convert(PR.ESTADO_CONTRIBUYENTE,\'US7ASCII\',\'WE8ISO8859P1\')AS ESTADO_CONTRIBUYENTE,convert(PR.CONDICION_DOMICILIO,\'US7ASCII\',\'WE8ISO8859P1\')AS CONDICION_DOMICILIO,convert(PR.ID_UBIGEO,\'US7ASCII\',\'WE8ISO8859P1\')AS ID_UBIGEO,convert(PR.TIPO_VIA,\'US7ASCII\',\'WE8ISO8859P1\')AS TIPO_VIA,convert(PR.NOMBRE_VIA,\'US7ASCII\',\'WE8ISO8859P1\')AS NOMBRE_VIA,convert(PR.NUMERO,\'US7ASCII\',\'WE8ISO8859P1\') AS NUMERO,convert(PR.MANZANA,\'US7ASCII\',\'WE8ISO8859P1\')AS MANZANA,convert(PR.KILOMETRO,\'US7ASCII\',\'WE8ISO8859P1\')AS KILOMETRO,convert(PR.LATITUD,\'US7ASCII\',\'WE8ISO8859P1\')AS LATITUD,convert(PR.LONGITUD,\'US7ASCII\',\'WE8ISO8859P1\')AS LONGITUD,convert(U.DISTRITO,\'US7ASCII\',\'WE8ISO8859P1\')AS DISTRITO,convert(U.PROVINCIA,\'US7ASCII\',\'WE8ISO8859P1\')AS PROVINCIA,convert(U.DEPARTAMENTO,\'US7ASCII\',\'WE8ISO8859P1\')AS DEPARTAMENTO,convert(U.POBLACION,\'US7ASCII\',\'WE8ISO8859P1\')AS POBLACION,convert(U.AREA_KM2,\'US7ASCII\',\'WE8ISO8859P1\')AS AREA_KM2  FROM PADRON_REDUCIDO PR INNER JOIN UBIGEO U  ON PR.ID_UBIGEO=U.ID_UBIGEO WHERE LATITUD IS NOT NULL AND U.DISTRITO=\'' . $P_DISTRITO . '\'';



    $stid = oci_parse($conn, $sql);
    oci_execute($stid);

    $result = '[';

    while (oci_fetch($stid)) {


        $LONGITUD = oci_result($stid, 'LONGITUD');
        $LATITUD = oci_result($stid, 'LATITUD');
        $RUC = oci_result($stid, 'RUC');
        $RAZON_SOCIAL = oci_result($stid, 'RAZON_SOCIAL');
        $TIPO_VIA = oci_result($stid, 'TIPO_VIA');
        $NOMBRE_VIA = oci_result($stid, 'NOMBRE_VIA');
        $NUMERO = oci_result($stid, 'NUMERO');

//DAREMOS FORMATO JSON PARA QUE EL JAVASCRIPT CLIENTE PUEDA INVOCAR Y USAR EN SUS PLUGINS AL SERVICIO
        $result .= '{position: {lng: ' . $LONGITUD . ', lat: ' . $LATITUD . '}, title: "' . $RUC . ' - ' . $RAZON_SOCIAL . ' - ' . $TIPO_VIA . ' ' . $NOMBRE_VIA . ' ' . $NUMERO . '"},';
    }
    $result = substr($result, 0, -1);
    $result .= '];';
    oci_free_statement($stid);
    oci_close($conn);
    return $result;
}

$server->register("GET_PADRON_DIRECCIONES", array("P_DISTRITO" => "xsd:string"), array("return" => "xsd:string"), "http://www.example.com", "", "", "", "Devolver datos del padron reducido");

// Run service
$server->service(file_get_contents('php://input'));
?>
```

### 3.- Crear la Aplicación Hibrida en APACHE CORDOVA y GOOGLE MAPS PLUGIN
Crearemos la siguiente clase index.html que sirva de Cliente WS y muestre el contenido del Padrón Reducido en Google Maps

```HTML
<!DOCTYPE html>
<html>

<head>
        <script type="text/javascript" src="js/starbucks_hawaii_stores.json"></script>
    <style type="text/css">
        #map_canvas {
            /* Must bigger size than 100x100 pixels */
            width: 100%;
            height: 500px;
        }

        button {
            padding: .5em;
            margin: .5em;
        }
    </style>


    <meta name="viewport" content="width=device-width">
    <script type="text/javascript" src="cordova.js"></script>



</head>

<body>

    <body>
        <div id="map_canvas">
            <span id="label" style="background-color:white;padding:.5em;font-size:125%;position:fixed;right:0;top:0;"></span>
            <button id="removeClusterBtn" style="position:fixed;bottom:0;right:0;padding:.5em;font-size:125%;">markerCluster.remove()</button>
        </div>
    
    </body>
</body>



<script src="jquery-3.3.1.min.js" type="text/javascript"></script>
<script src="jquery.soap.js" type="text/javascript"></script>
<script type="text/javascript">

    //Android, IOS
    document.addEventListener("deviceready", function () {
        

var mapDiv = document.getElementById("map_canvas");
var options = {
    'camera': {
        'target': data[0].position,
        'zoom': 3
    }
};
var map = plugin.google.maps.Map.getMap(mapDiv, options);
map.on(plugin.google.maps.event.MAP_READY, onMapReady);
});


function onMapReady() {
var map = this;

var label = document.getElementById("label");

//------------------------------------------------------
// Create a marker cluster.
// Providing all locations at the creating is the best.
//------------------------------------------------------
map.addMarkerCluster({
  //debug: true,
  //maxZoomLevel: 5,
  markers: data,
  icons: [
      {min: 2, max: 100, url: "./img/blue.png", anchor: {x: 16, y: 16}},
      {min: 100, max: 1000, url: "./img/yellow.png", anchor: {x: 16, y: 16}},
      {min: 1000, max: 2000, url: "./img/purple.png", anchor: {x: 24, y: 24}},
      {min: 2000, url: "./img/red.png",anchor: {x: 32,y: 32}},
  ]
}, function (markerCluster) {

    //-----------------------------------------------------------------------
    // Display the resolution (in order to understand the marker cluster)
    //-----------------------------------------------------------------------
    markerCluster.on("resolution_changed", function (prev, newResolution) {
        var self = this;
        label.innerHTML = "<b>zoom = " + self.get("zoom") + ", resolution = " + self.get("resolution") + "</b>";
    });
    markerCluster.trigger("resolution_changed");



    //----------------------------------------------------------------------
    // Remove the marker cluster
    // (Don't remove/add repeatedly. This is really bad performance)
    //----------------------------------------------------------------------
    var removeBtn = document.getElementById("removeClusterBtn");
    removeBtn.addEventListener("click", function () {
        

        var mapDiv = document.getElementById("map_canvas");
        var options = {
            'camera': {
                'target': data[0].position,
                'zoom': 3
            }
        };
        var map = plugin.google.maps.Map.getMap(mapDiv, options);
        map.on(plugin.google.maps.event.MAP_READY, onMapReady);
        }, {
      once: true
    });

    //------------------------------------
    // If you tap on a marker,
    // you can get the marker instnace.
    // Then you can do what ever you want.
    //------------------------------------
    var htmlInfoWnd = new plugin.google.maps.HtmlInfoWindow();
    markerCluster.on(plugin.google.maps.event.MARKER_CLICK, function (position, marker) {
      var html = [
        "<div style='width:250px;min-height:100px'>",
        "<img src='img/starbucks_logo.gif' align='right'>",
        "<strong>" + marker.get("title")  + "</strong>"];
      
      html.push("</div>");
      htmlInfoWnd.setContent(html.join(""));
      htmlInfoWnd.open(marker);
    });


});

}
</script>

</html>
``` 

![GitHub Logo](/1880.png)
![GitHub Logo](/31.png)
![GitHub Logo](/detalle.png)

