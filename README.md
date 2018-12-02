# loopback-trasaction-multiple-models-and-promise
example of loopback trasaction with multiple models and promise


Example of my model controller with a trasacction with promise:

```javascript
'use strict';

module.exports = function (Grupos) {

    var app = require('../../server/server');

    Grupos.crear_grupo = function (req, res, options, data, cb) {

        var errores = [];
        var Modelos = app.models.Modelos;
        var Permisos = app.models.Permisos;

        var Validator = require('jsonschema').Validator;
        var v = new Validator();

        var schema = {
            "type": "object",
            "properties": {
                "nombre": { "type": "string" },
                "descripcion": { "type": "string" },
                "modelos": {
                    "type": "array",
                    "items": {
                        "properties": {
                            "nombre": { "type": "string" },
                            "permisos": {
                                "type": "array",
                                "items": {
                                    "properties": {
                                        "nombre": { "type": "string" }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        };

        var validacion = v.validate(data, schema);

        if (validacion.errors.length > 0) {
            console.log(validacion);
            validacion.errors.forEach(element => {
                errores.push(element.stack);
            });
            res.status(400).send(errores);
        } else {

            var promesasM = [];
            var promesasP = [];
            var transacciones = [];

            Grupos.beginTransaction({ isolationLevel: Grupos.Transaction.READ_COMMITTED }).then(function (tx) {
                // Now we have a transaction (tx)

                // creacion de grupo

                Grupos.create(data, { transaction: tx }).then(function (grupo) {
                    data.modelos.forEach(element => {
                        element.id_grupo = grupo.id;
                        // creacion de modelos
                        promesasM.push(
                            Modelos.create(element, { transaction: tx }).then(function (modelo) {
                                element.id_modelo = modelo.id;
                                return element;
                            }).catch(function (err) {
                                console.log(err);
                                return null;
                            }));
                    });

                    Promise.all(promesasM).then(values => {
                        if (values.includes(null)) {
                            tx.rollback().then(function (data) {
                                console.log(data);
                                res.status(500).send("error");
                            }).catch(function (err) {
                                res.status(500).send(err);
                            });
                        } else {
                            values.forEach(modelo => {
                                modelo.permisos.forEach(element => {
                                    element.id_modelo = modelo.id_modelo;
                                    // creacion de permisos
                                    promesasP.push(Permisos.create(element, { transaction: tx }).then(function (permiso) {
                                        return element;
                                    }).catch(function (err) {
                                        console.log(err);
                                        return null;
                                    }));
                                });
                            });

                            Promise.all(promesasP).then(values => {
                                if (values.includes(null)) {
                                    tx.rollback().then(function (data) {
                                        console.log(data);
                                        res.status(500).send("error");
                                    }).catch(function (err) {
                                        res.status(500).send(err);
                                    });
                                } else {
                                    tx.commit().then(function (data) {
                                        console.log(data);
                                        res.status(201).send("OK")
                                    }).catch(function (err) {
                                        res.status(500).send(err);
                                    });
                                }
                            });
                        }
                    });


                }).catch(function (err) {
                    console.log(err);
                    var error = err;
                    tx.rollback().then(function (data) {
                        console.log(data);
                        errores.push(error.message);
                        res.status(error.statusCode).send(errores);
                    }).catch(function (err) {
                        res.status(500).send(err);
                    });
                });
            }).catch(function (err) {
                console.log(err);
                res.status(500).send(err);
            });
        }

    };

    Grupos.remoteMethod(
        'crear_grupo', {
            http: {
                path: '/crear_grupo',
                verb: 'post'
            },

            accepts: [
                { arg: 'req', type: 'object', 'http': { source: 'req' } },
                { arg: 'res', type: 'object', 'http': { source: 'res' } },
                { arg: "options", type: "object", http: "optionsFromRequest" },
                { arg: 'data', type: "object" }
            ],
            returns: {
                arg: 'resultado',
                type: 'string'
            }
        }
    );

};
```
