<template>
  <v-app-bar app flat dense color="blue lighten-4" absolute>
    <v-toolbar-items>

      <v-btn-toggle borderless tile background-color="blue lighten-4">
        <v-btn icon disabled>
          <v-icon>mdi-hand-right</v-icon>
        </v-btn>

        <v-btn icon disabled>
          <v-icon>mdi-magnify-plus</v-icon>
        </v-btn>

        <v-btn icon disabled>
          <v-icon>mdi-magnify-minus</v-icon>
        </v-btn>
      </v-btn-toggle>

      <!-- <v-combobox
              solo
              flat
              v-model="select"
              :items="items"
              label="Subsets"
              chips
              background-color="blue lighten-4"
      >
        <template v-slot:selection="data">
          <v-chip
                  :key="JSON.stringify(data.item.text)"
                  v-bind="data.attrs"
                  :input-value="data.selected"
                  :disabled="data.disabled"
                  @click:close="data.parent.selectItem(data.item.text)"
          >
            <v-avatar
                    class="accent white--text"
                    left
            >
              <v-icon>{{ data.item.icon }}</v-icon>
            </v-avatar>
            <v-icon left>{{ data.item.icon }}</v-icon>{{ data.item.text }}
          </v-chip>
        </template>
      </v-combobox> -->

      <!-- <v-btn-toggle
                    @change="subset_mode_changed"
                    mandatory
                    borderless
                    tile
                    background-color="blue lighten-4">
        <v-btn>
          <v-icon>mdi-checkbox-blank-circle</v-icon>
        </v-btn>
        <v-btn>
          <v-icon>mdi-set-all</v-icon>
        </v-btn>
        <v-btn>
          <v-icon>mdi-set-center</v-icon>
        </v-btn>
        <v-btn>
          <v-icon>mdi-set-left-right</v-icon>
        </v-btn>
        <v-btn>
          <v-icon>mdi-set-center-right</v-icon>
        </v-btn>
      </v-btn-toggle> -->

      <g-subset-select></g-subset-select>
      <g-subset-mode></g-subset-mode>

      <v-btn-toggle borderless tile background-color="blue lighten-4">
        <v-btn icon disabled>
          <v-icon>mdi-contrast-box</v-icon>
        </v-btn>
      </v-btn-toggle>

      <v-btn-toggle borderless tile background-color="blue lighten-4">
        <v-btn icon disabled>
          <v-icon>mdi-cube-outline</v-icon>
        </v-btn>
      </v-btn-toggle>

      <v-divider vertical></v-divider>

      <template v-for="tool in tools">
        <jupyter-widget :widget="tool" />
      </template>

    </v-toolbar-items>
  </v-app-bar>
</template>
