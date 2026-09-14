```
                                    ┌──────────────────────┐
                                    │ Configuration        │
                                    │ validation           │
                                    └──────────┬───────────┘
                                               │
                                               ▼
                                        Component
                                         validation
                                               │
                                               ▼
                                        Source validation
                                               │
                                               ▼
                                        Group by filename
                                             date
                                               │
                                               ▼
                                      ┌─────────────────┐
                                      │ Date processing │
                                      └────────┬────────┘
                                               │
                                    ┌──────────┼──────────┐
                                    ▼          ▼          ▼
                              Destination   Source      Neither
                               archive      archive
                                 exists      exists
                                    │          │          │
                                    ▼          ▼          ▼
                                 Verify     Verify      Create
                                    │          │          │
                                 PASS       PASS       Verify
                                    │          │          │
                                    ▼          ▼          ▼
                                 Delete     Move       Move
                                 source      archive     archive
                                               │          │
                                               ▼          ▼
                                            Verify      Verify
                                            presence    contents
                                               │          │
                                               └────┬─────┘
                                                    ▼
                                              Delete source
                                                    │
                                                    ▼
                                           Component summary
                                                    │
                                                    ▼
                                           Global summary
                                                    │
                                                    ▼
                                             Exit status
```