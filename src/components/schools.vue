<template>
    <div>
      <div class="flex-container">
        <HomeSchool :recs=elementary heading="ELEMENTARY" />
        <HomeSchool :recs=middle heading="MIDDLE" />
        <HomeSchool :recs=high heading="HIGH" />
        <div class="report-record-highlight flex-item text-center" v-if="zone">
          <svg class="icon icon-bus"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#icon-bus"></use></svg>
          <h2>Your Transportation Zone is</h2>
          <h1>{{zone}}</h1>
        </div>
      </div>
        
      <div v-if="magnet">
        <table>
            <caption>Magnet Schools</caption>
            <thead>
                <tr>
                    <th>School</th>
                    <th>Address</th>
                    <th>Grades</th>
                    <th class="col-responsive text-right">Distance</th>
                </tr>
            </thead>
            <tbody>
                <tr v-for="(item, index) in magnet">
                    <td>
                        {{item.name}}{{ item.num | getAsterik }}
                        <span class="magnet-focus">{{item.mag_focus}}</span>
                    </td>
                    <td>
                        <a href="javascript:void(0)" v-on:click="locationClick(magnet[index])">
                                {{item.address}}, {{item.city}}</a>
                    </td>
                    <td>
                      {{item.grade_level}}
                    </td>
                    <td class="nowrap col-responsive text-right">
                        {{ item.distance | distance }}
                    </td>
                </tr>
            </tbody>
        </table>
      </div>

      <div class="report-moreinfo">
        <p>
            Transportation eligibility is determined by the transportation zone in which you live. The county is divided into three transportation zones: violet, blue and green. Magnet schools are assigned a zone to serve with transportation. If the distance
            is to the magnet greater than 5 miles, you may be assigned to a shuttle stop location. Some programs (*) provide county-wide transportation, meaning that regardless of your zone, you would receive transportation. For more information please
            contact Charlotte-Mecklenburg School Transportation Services at (980) 343-6715.
        </p>
        <h5>For more information, please visit:</h5>
        <ul class="list-unstyled">
            <li><a href="http://www.cms.k12.nc.us/" target="_blank"  rel="noopener">Charlotte-Mecklenburg Schools</a></li>
        </ul>
      </div>
  </div>
</template>

<script>
import jsonToURL from "../js/jsontourl";
import format from "format-number";
import HomeSchool from './schools_home.vue'

export default {
  name: "schools",

  components: {
    HomeSchool
  },

  data: function() {
    return {
      magnet: [],
      elementary: [],
      middle: [],
      high: [],
      zone: null
    };
  },

  computed: {
    selected () {
      return this.$store.getters.selected
    },
    show () {
      return this.$store.getters.show
    }
  },

  watch: {
    selected: "getResults",
    show: "getResults"
  },

  filters: {
    distance: function(dist) {
      return format({
        truncate: 1,
        suffix: " miles"
      })(dist / 5280);
    },
    getAsterik: function(id) {
      if (
        [4429, 8364, 5532, 8482, 7405, 7496, 5520].indexOf(Number(id)) !== -1
      ) {
        return "*";
      } else {
        return "";
      }
    }
  },

  mounted: function() {
    this.getResults();
  },

  methods: {       
    getResults: function() {
      let _this = this;
      if (
        _this.selected.lnglat &&
        _this.show.indexOf("schools") !== -1
      ) {

        // school assignments and transportation zone
        fetch(`https://mcmap.org/api/nearest/v1/cms_parcels/${_this.selected.lnglat[0]},${_this.selected.lnglat[1]}/4326?columns=elem_num,high_zone,midd_num,high_num,schl_other&limit=1`)
          .then( response => response.json())
          .then( response => {
            // set transportation zone
              _this.zone = response[0].high_zone;

            // get schools
            let schlnums = [response[0].elem_num, response[0].midd_num, response[0].high_num];
            if (response[0].schl_other) {              
              schlnums = schlnums.concat(response[0].schl_other.split(',').map(Number));
            }
            return fetch(`https://mcmap.org/api/query/v1/cms_schools?columns=city,zipcode::int,address,name,type,grade_level,ST_Distance(geom,ST_Transform(GeomFromText('POINT( ${Number(_this.selected.lnglat[0]         )} ${Number(
              _this.selected.lnglat[1]
            )} )',4326), 2264)) as distance,st_x(st_transform(geom, 4326)) as lng, st_y(st_transform(geom, 4326)) as lat&filter=num in(${schlnums.join()})`)
          })
          .then( response => response.json())
          .then( schools => {
            _this.elementary = schools.filter(item => item.type === 'ELEMENTARY').sort((a,b) => {if (a.grade_level < b.grade_level) {return 1} else {return -1}})
            _this.middle = schools.filter(item => item.type === 'MIDDLE').sort((a,b) => {if (a.grade_level < b.grade_level) {return 1} else {return -1}})
            _this.high = schools.filter(item => item.type === 'HIGH').sort((a,b) => {if (a.grade_level < b.grade_level) {return 1} else {return -1}})
          })
          .catch(function(ex) {
            console.log("parsing failed", ex);
          });

        // magnet schools
        fetch(`https://mcmap.org/api/query/v1/cms_schools?${jsonToURL({
          filter: "magnet in ('Full', 'Partial')",
          columns: `num,city,zipcode::int,address,name,type,grade_level,mag_focus,st_x(st_transform(geom, 4326)) as lng, st_y(st_transform(geom, 4326)) as lat,magnet,ST_Distance(geom,ST_Transform(GeomFromText('POINT( ${Number(_this.selected.lnglat[0]         )} ${Number(
              _this.selected.lnglat[1]
            )} )',4326), 2264)) as distance`,
            sort: 'distance'
            })}`)
          .then( response => response.json())
          .then( response => {
            _this.magnet = response;
          })
          .catch(function(ex) {
            console.log("parsing failed", ex);
          });

      }
    },
    locationClick: function(rec) {
      this.$store.commit("poi", {
        lnglat: [rec.lng, rec.lat],
        address: rec.address,
        label: rec.name ? rec.name : rec.schlname
      })
    }
  }
};
</script>

<style scoped>
.flex-container {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
}
.flex-item {
  flex-wrap: wrap;
}
.report-record-highlight {
  margin: 20px;
  text-align: center;
}
.magnet-focus {
  display: block;
  font-size: 0.85em;
  color: #6b6b6b;
}
</style>