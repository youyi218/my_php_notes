## Motif

Un collègue m'a demandé de lui donner des données de certaine période pour faire son propre étude. je me suis dit pourquoi pas mettre en oeuvre une petite fonctionnalité pour télécharger des données en format csv. Voila

## Options

* A GET request pour un URL (pas de sécurité)
* AJAX pour lancer la recherche et récupérer des données, puis JavaScript s'occupe de la création du fichier .csv (pas très naturel de utiliser JavaScript pour ce genre de tache)
* AJAX pour lancer la recherche, puis retourne un URL, cet URL pour lancer un autre PHP script, qui connaît l'emplacement du fichier (sauvegardé dans la session), ajouter la tête et puis lancer le téléchargement

Je choisis la troisième option, la requête de base de données demande quelques secondes, donc je pense AJAX est approprié, puis comme j'ai dit, plus naturel de récupérer directement le ressource via PHP (à discuter, je ne suis pas très confident)

## Codes:



```php
/**
     * Returns the given Flowbox download Wifi data page HTML
     * 
     * @Route("/downloadWifiData")
     * @param integer $id
     * @param integer $dateStart
     * @param integer $dateEnd
     */
    public function launchFlowboxWifiDataDownload(Request $req)
    {
        $id = $req->get('id');
        $timeStart = $req->get('timeStart');
        $timeEnd = $req->get('timeEnd');
        $Flowbox = $this->flowboxQuery->getFlowboxUsingId($id);
        $filename = 'WIFIDATA_FLOWBOX' . $Flowbox->getNumber() . '_FROM_' . $timeStart . '_TO_' . $timeEnd . '.csv';
        if (null == $Flowbox) {
            throw $this->createNotFoundException('Cette Flowbox n\'existe pas');
        }
        // wifi_mac, wifi_mvt, wifi_power, wifi_datestart, wifi_dateend, STOP1.stop_name, STOP2.stop_name
        $wifi_a = $this->flowboxQuery->getWifiUsingFlowboxNumberWithDateStartAndDateEnd($timeStart, $timeEnd, $Flowbox->getNumber());
        if (count($wifi_a) == 0) {
            return new Response("Données non trouvées", 404);
        }
        $csvWifi = fopen($filename, 'w');
        fputcsv($csvWifi, array('wifi_mac', 'wifi_mvt', 'wifi_power', 'wifi_datestart', 'wifi_dateend', 'wifi_departure', 'wifi_terminal'));
        foreach ($wifi_a as $line) {
            fputcsv($csvWifi, $line);
        }
        fclose($csvWifi);
        $this->session->set('wifiDownloadUrl', $filename);
        return new Response("Données trouvées", 200);
    }

    /**
     * Return the download URL
     * 
     * @Route("/retrieveWifiFile")
     */
    public function retrieveWifiDownloadURL()
    {
        ignore_user_abort(true);
        $filename = $this->session->get('wifiDownloadUrl');
        header('Content-Type: text/csv; charset=utf-8');
        header('Content-Disposition: attachment; filename=wifi.csv');
        readfile($filename);
        if (connection_aborted()) {
            unlink($filename);
        }
        // To delete the file the server side
        unlink($filename);
        exit;
    }

```



```javascript
if (document.getElementById("checkboxWifi").checked) {
        // start AJAX to launch wifi and gps query and download
        queryDownloadWifi = jQuery.ajax({
            method: "POST",
            url: "/index.php/Flowbox/downloadWifiData",
            data: {
                'id': flowboxId,
                'timeStart': timeStart,
                'timeEnd': timeEnd,
            },
            success: function (data) {
                window.open('/index.php/Flowbox/retrieveWifiFile');
            }
        });
    }
```