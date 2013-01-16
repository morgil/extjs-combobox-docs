How do I ComboBox?
==================

ComboBoxen sind die ExtJS-Version von Dropdown-Feldern. In Formularen verhält sich eine ComboBox genau so wie man es erwartet, sie zeigt den Wert aus dem gesetzten displayField an und hält als Wert den Feld aus dem valueField.

## Comboboxen in Grids

Das Verhalten von Combobxen in Grids ist immer noch logisch, aber nicht intuitiv: Der Grid zeigt dann den Wert des valueFields an, statt den angezeigten Wert den der User angeklickt hat. Das liegt daran dass die Combobox eigentlich nur ein View-Element ist, das nach der Auswahl nur den Wert aus dem valueField an den Grid (das Model) weitergibt, der Grid erfährt nicht welches displayField dazu gehört.

Wie löst man das jetzt? Die Gridspalte in der die Combobox definiert ist, hat die Eigenschaft renderer, eine Funktion um die Darstellung im Grid zu manipulieren. Wenn die Combobox also einen ganz simplen Store hat der aus einer Variablen gefüttert wird, lässt man den renderer einfach diese Variable durchsuchen und das gewünschte Feld zurückgeben. Beispiel für einen normalen Object-Store:

    renderer: function(value) {
        return the_store_object[value];
    }

Ist der Store komplexer und holt sich seine Daten erst per JSON vom Server, ist das schwieriger. Man kann nicht einfach in den Renderer eine entsprechende Abfrage an den Server einbauen, weil die Ext.Direct-Abfragen asynchron laufen. Man kann aber im load-Listener des Combobox-Stores die Ergebnisse cachen und diese dann im Renderer wieder abfragen.

    load: function(me, records) {
        if (!box_cache) {
            box_cache = {};
        }
        for (var key in records) {
            if (!records.hasOwnProperty(key)) {
                continue;
            }
            box_cache[records[key].data.data_value] = records[key].data.display_value;
        }
    }
    [...]
    renderer: function(value) {
        if (!box_cache || !box_cache[value]) {
            return value;
        }
        return box_cache[value];
    }


Und um das Ganze noch ein bisschen schwieriger zu machen, kann der User auch nach dem Eingeben von einem (gültigen) Wert in der Combobox einfach nur woanders hin klicken. Dann wird die Combobox auch zugemacht, aber wenn die nicht die Option forceSelection aktiv ist wird dann der eingegebene Wert zurückgegeben statt dessen ID.
Wenn man forceSelection nicht haben will, gibt es aber eine Möglichkeit deren Verhalten zu simulieren: Das blur-Event der Combobox, das vor dem edit-Event ausgelöst wird, kümmert sich folgendermaßen darum:

    blur: function(field, eOpts) {
        if (field.value) { // Ein Eintrag aus der Liste wurde ausgewählt
            return;
        }
        var rec = field.findRecord(field.displayField, field.getRawValue());
        if (rec) { // Der Eintrag ist vorhanden, wurde aber nicht ausgewählt
            field.setValue(rec.get(field.valueField));
        }
    }

Jetzt steht im Feld immer die ID des ausgewählten Werts falls er existiert, und der Wert selbst falls er nicht existiert. Falls mehrere Werte mit gleichem Namen existieren, muss man sich selbst entscheiden was man damit tun will.

## Der Store einer Combobox wird erst beim zweiten Anwählen geladen

Situation: Die Box hat einen afterrender-Listener, in der `this.focus()` aufgerufen wird. Außerdem hat sie einen Store in dem `autoload: true` gesetzt sein mag, der wird aber erst geladen wenn der User die Box deselektiert und wieder anwählt.

Lösung: `focus()` hat zwei Parameter, der zweite verzögert die Befehlsausführung um 10ms - genug dass der Store in der Zwischenzeit geladen wird. Also ändert man den Aufruf auf `this.focus(false, true)` und der Store steht von Anfang an zur Verfügung.