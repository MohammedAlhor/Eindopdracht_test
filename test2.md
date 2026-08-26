Ja, precies. Mijn laatste uitleg was te versimpeld.

De **nodestructuur is de structuur van het hele document**, niet alleen van de tabel:


root
└── section
    ├── heading
    ├── paragraph
    │   └── sentence-nodes
    ├── table
    │   └── table_row
    │       └── cell
    │           └── sentence-node
    ├── list
    └── paragraph
        └── sentence-nodes


De tabel wordt dus als nieuw nodetype **tussen de bestaande geparste tekstnodes geplaatst**, in de oorspronkelijke documentvolgorde.

Een chunk kan vervolgens node-ID’s bevatten van:

- gewone tekst;
- tabelcellen;
- eventueel tekst vóór en na de tabel.

De citation verwijst naar één relevante node. Bij een tabelantwoord kan dat een celnode zijn; bij uitleg kan het een gewone sentence-node zijn. React toont vervolgens de bijbehorende chunk met de omliggende inhoud.

Het notebook demonstreert nu alleen het **tabelonderdeel** van deze grotere nodestructuur. Het is dus een tabelspike, niet een vervanging van de bestaande documentparser. Dat moet je ook zo aan je collega’s uitleggen.
